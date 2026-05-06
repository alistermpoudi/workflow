# Phase 3 — Tâche 3.2 : SkillCurator.py

## Objectif

Créer `SkillCurator.py` — le **consolidateur LLM des skills accumulés**. `ExecutionLoop` crée des skills bruts à chaque retry réussi (Phase 3.1). Sans curation, on accumule du bruit, des doublons, des skills trop spécifiques. Le Curator tourne périodiquement (manuel ou daemon) et :

- **Déduplique** les skills similaires
- **Consolide** les patterns récurrents (≥2 skills similaires → 1 skill généralisé)
- **Promeut au global** les skills locaux utilisés ≥3 fois sur ≥2 projets différents
- **Archive** les skills jamais utilisés depuis 90+ jours

> **Pilier load-bearing #2 (deuxième moitié).** Sans Curator, la boucle d'apprentissage de Workflow tombe en panne dans les 3 mois — le bruit noie les vrais patterns.

## Dépendances

- Phase 1 ✅ (`SkillManager`)
- Tâche 2.1 ✅ (`LLMProvider` avec rôle `curator`)
- Tâche 3.1 ✅ (`ExecutionLoop` qui crée les skills bruts)

## Fichiers à Créer

- `src/workflow/core/skill_curator.py` [CRÉER]
- `tests/unit/test_skill_curator.py` [CRÉER]

## Fréquence de Run

- **Manuel** : `workflow curate` ou via MCP `workflow_curate_skills()`
- **Auto (daemon — Phase 7)** : `DaemonHeartbeat` lance le Curator quotidiennement si ≥10 nouveaux skills depuis la dernière run
- **Trigger événementiel** : à la complétion d'une version (≥10 skills créés pendant la version)

## Implémentation

```python
# src/workflow/core/skill_curator.py
import json
from pathlib import Path
from datetime import datetime, timezone, date as date_cls, timedelta

from workflow.core.skill_manager import SkillManager
from workflow.llm.llm_provider import LLMProvider
from workflow.llm.prompt_builder import PromptBuilder

ARCHIVE_THRESHOLD_DAYS = 90
PROMOTE_USAGE_THRESHOLD = 3
PROMOTE_PROJECT_THRESHOLD = 2


class SkillCurator:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.project_root = Path(project_root)
        self.skills = SkillManager(project_root)
        self.llm = llm
        self.io = io

    async def run(self, dry_run: bool = False) -> dict:
        """
        Lancer une session de curation complète.
        Retourne un rapport : {deleted, consolidated, promoted, archived}.
        """
        self.io.print_header('SkillCurator — Consolidation')
        all_skills = self.skills.list_all()
        self.io.print_info(f'Analyse de {len(all_skills)} skills...')

        if not all_skills:
            self.io.print_warning('Aucun skill à curer.')
            return {'deleted': [], 'consolidated': [], 'promoted': [], 'archived': []}

        # Étape 1 : pré-filtrage local (sans LLM) — détecter doublons exacts par fingerprint
        prefiltered = self._prefilter_duplicates(all_skills)

        # Étape 2 : LLM consolide les skills (clusterise les similaires)
        consolidation_plan = await self._llm_consolidate(prefiltered['kept'])

        # Étape 3 : appliquer le plan
        report = {
            'deleted': prefiltered['exact_duplicates'] + consolidation_plan.get('delete', []),
            'consolidated': consolidation_plan.get('consolidate', []),
            'promoted': self._select_for_promotion(prefiltered['kept']),
            'archived': self._select_for_archive(prefiltered['kept']),
        }

        if dry_run:
            self._print_report(report, dry_run=True)
            return report

        await self._apply(report)
        self._print_report(report, dry_run=False)
        return report

    # ─── Pré-filtrage local ───────────────────────────────────────────

    def _prefilter_duplicates(self, skills: list[dict]) -> dict:
        """Détecter les doublons exacts par fingerprint du contenu"""
        seen: dict[str, dict] = {}
        exact_duplicates: list[str] = []
        kept: list[dict] = []

        for skill in skills:
            fp = self._fingerprint(skill)
            if fp in seen:
                # Garder celui avec le plus d'usage_count
                if skill.get('usage_count', 0) > seen[fp].get('usage_count', 0):
                    exact_duplicates.append(seen[fp]['name'])
                    seen[fp] = skill
                else:
                    exact_duplicates.append(skill['name'])
            else:
                seen[fp] = skill

        kept = list(seen.values())
        return {'kept': kept, 'exact_duplicates': exact_duplicates}

    def _fingerprint(self, skill: dict) -> str:
        """Fingerprint stable pour détecter les doublons exacts"""
        import hashlib
        content = (skill.get('description', '') + skill.get('content', ''))[:1000]
        normalized = ' '.join(content.lower().split())
        return hashlib.md5(normalized.encode()).hexdigest()[:16]

    # ─── Consolidation LLM ────────────────────────────────────────────

    async def _llm_consolidate(self, skills: list[dict]) -> dict:
        """Demander au LLM (rôle curator) de proposer une consolidation"""
        # Charger le contenu complet des skills (pas juste les métadonnées)
        full_skills = []
        for s in skills:
            try:
                content = Path(s['path']).read_text(encoding='utf-8')
                _, body = self.skills._parse_frontmatter(content)
                full_skills.append({**s, 'content': body})
            except OSError:
                continue

        if not full_skills:
            return {}

        # Batcher si > 50 skills (LLM context limit)
        batch_size = 50
        consolidation = {'delete': [], 'consolidate': [], 'promote_global': [], 'archive': []}

        for i in range(0, len(full_skills), batch_size):
            batch = full_skills[i:i + batch_size]
            prompt = PromptBuilder.curator_consolidate(batch)
            response = await self.llm.ask(prompt, role='curator')
            try:
                # Extraire le JSON de la réponse (parfois entouré de markdown)
                json_text = self._extract_json(response)
                batch_plan = json.loads(json_text)
                for key in consolidation:
                    consolidation[key].extend(batch_plan.get(key, []))
            except (json.JSONDecodeError, ValueError) as e:
                self.io.print_warning(f'Batch {i // batch_size} : JSON invalide ({e})')
                continue

        return consolidation

    @staticmethod
    def _extract_json(text: str) -> str:
        """Extraire un JSON d'une réponse LLM (parfois entouré de ```json ... ```)"""
        text = text.strip()
        if text.startswith('```'):
            lines = text.split('\n')
            return '\n'.join(lines[1:-1])
        return text

    # ─── Promotion automatique ────────────────────────────────────────

    def _select_for_promotion(self, skills: list[dict]) -> list[str]:
        """
        Sélectionner les skills locaux à promouvoir au global.
        Critères : usage_count >= 3 ET utilisé sur ≥2 projets différents (track via metadata).
        """
        promoted = []
        for s in skills:
            if s.get('is_global'):
                continue  # Déjà global
            if s.get('usage_count', 0) >= PROMOTE_USAGE_THRESHOLD:
                # Approximation : si learned_from couvre plusieurs TASK, considérer multi-projet
                projects = self._infer_projects_used(s)
                if len(projects) >= PROMOTE_PROJECT_THRESHOLD:
                    promoted.append(s['name'])
        return promoted

    def _infer_projects_used(self, skill: dict) -> set:
        """Stub — Phase 9 améliorera avec un vrai tracking des projets utilisateurs"""
        # Pour le MVP : suppose que chaque "learned_from" = un projet différent
        # (acceptable car learned_from est souvent un ID de tâche unique)
        return {skill.get('learned_from', '')}

    # ─── Archivage automatique ────────────────────────────────────────

    def _select_for_archive(self, skills: list[dict]) -> list[str]:
        """Sélectionner les skills jamais utilisés depuis 90+ jours"""
        cutoff = date_cls.today() - timedelta(days=ARCHIVE_THRESHOLD_DAYS)
        archived = []
        for s in skills:
            last_used = s.get('last_used')
            learned_at = s.get('learned_at')
            try:
                effective_date = date_cls.fromisoformat(last_used or learned_at or '2000-01-01')
            except ValueError:
                continue
            if effective_date < cutoff and s.get('usage_count', 0) == 0:
                archived.append(s['name'])
        return archived

    # ─── Application ──────────────────────────────────────────────────

    async def _apply(self, report: dict):
        """Appliquer les changements proposés"""
        # Supprimer
        for name in report['deleted']:
            self._delete_skill(name)

        # Consolider
        for cluster in report['consolidated']:
            self._create_consolidated_skill(cluster)
            for old_name in cluster.get('merge_from', []):
                self._delete_skill(old_name)

        # Promouvoir au global
        for name in report['promoted']:
            self._promote_to_global(name)

        # Archiver
        for name in report['archived']:
            self._archive_skill(name)

    def _delete_skill(self, name: str):
        all_skills = self.skills.list_all()
        target = next((s for s in all_skills if s['name'] == name), None)
        if target:
            Path(target['path']).unlink(missing_ok=True)

    def _create_consolidated_skill(self, cluster: dict):
        self.skills.create_skill({
            'name': cluster['new_name'],
            'description': cluster.get('description', 'Skill consolidé par Curator'),
            'tags': ['consolidated'],
            'body': cluster.get('content', ''),
            'learned_from': 'curator',
        }, scope='global')

    def _promote_to_global(self, name: str):
        all_skills = self.skills.list_all()
        target = next((s for s in all_skills if s['name'] == name), None)
        if not target:
            return
        # Lire le fichier local et recréer en global
        content = Path(target['path']).read_text(encoding='utf-8')
        global_path = self.skills.global_skills_dir / f'{name}.md'
        global_path.parent.mkdir(parents=True, exist_ok=True)
        global_path.write_text(content, encoding='utf-8')
        Path(target['path']).unlink()

    def _archive_skill(self, name: str):
        """Archiver = déplacer vers ~/.workflow/skills/_archive/"""
        all_skills = self.skills.list_all()
        target = next((s for s in all_skills if s['name'] == name), None)
        if not target:
            return
        archive_dir = Path.home() / '.workflow' / 'skills' / '_archive'
        archive_dir.mkdir(parents=True, exist_ok=True)
        Path(target['path']).rename(archive_dir / f'{name}.md')

    # ─── Reporting ────────────────────────────────────────────────────

    def _print_report(self, report: dict, dry_run: bool):
        prefix = '[DRY-RUN] ' if dry_run else ''
        self.io.print(f'{prefix}Suppressions : {len(report["deleted"])}')
        self.io.print(f'{prefix}Consolidations : {len(report["consolidated"])}')
        self.io.print(f'{prefix}Promotions globales : {len(report["promoted"])}')
        self.io.print(f'{prefix}Archivages : {len(report["archived"])}')

        for cluster in report['consolidated']:
            self.io.print(f'  → {cluster["new_name"]} ← {", ".join(cluster.get("merge_from", []))}')
```

## Intégration avec les autres modules

**`ExecutionLoop` (Phase 3.1)** — crée des skills "bruts" après retry réussi. Le Curator nettoie après.

**`MCPServer` (Phase 6)** — expose `workflow_curate_skills(dry_run=true|false)` pour usage manuel.

**`DaemonHeartbeat` (Phase 7)** — lance auto si ≥10 skills depuis la dernière run :

```python
async def maybe_run_curator(self):
    last_run = await self.memory.get_last_curator_run()
    skills_since = self.skills.count_created_since(last_run)
    if skills_since >= 10:
        curator = SkillCurator(self.project_root, self.llm, self.io)
        await curator.run(dry_run=False)
        await self.memory.set_last_curator_run(datetime.now(timezone.utc))
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `_prefilter_duplicates()` détecte les doublons exacts par fingerprint | ⬜ |
| 2 | `_llm_consolidate()` batche par 50 skills si > 50 | ⬜ |
| 3 | Le Curator utilise `role='curator'` (pas `reasoning` ni `fast`) | ⬜ |
| 4 | `_select_for_promotion()` ne promeut qu'avec usage_count ≥ 3 et ≥2 projets | ⬜ |
| 5 | `_select_for_archive()` ne sélectionne que les skills 90j+ avec usage=0 | ⬜ |
| 6 | `dry_run=True` ne modifie aucun fichier, retourne le plan | ⬜ |
| 7 | `_apply()` supprime les fichiers physiques des skills `deleted` | ⬜ |
| 8 | `_create_consolidated_skill()` crée un nouveau skill `global` avec tag `consolidated` | ⬜ |
| 9 | `_archive_skill()` déplace vers `~/.workflow/skills/_archive/` (ne supprime pas) | ⬜ |
| 10 | Tests : 6 scénarios (dups, consolidation mock LLM, promotion, archive, dry-run, batch ≥50) | ⬜ |

## Notes d'Implémentation

- Le LLM peut halluciner des suggestions de consolidation. Filtrer : ne consolider que si ≥2 skills sources réels existent.
- L'archivage est **réversible** (déplacement, pas suppression). Permet de récupérer un skill mis de côté à tort.
- Pour Phase 9 (`WorkflowLibrary`) : tracker explicitement quel projet a utilisé quel skill (pas juste `learned_from`) pour la décision de promotion.
