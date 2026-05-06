# Phase 9 — Tâche 9.4 : ProjectIngester (workflow learn-from)

## Objectif

Créer `ProjectIngester.py` — le pipeline qui **absorbe les patterns d'un projet existant** pour les transformer en skills réutilisables. Le dev pointe son ancien projet (`workflow learn-from /path/to/project`), tag ce qu'il veut absorber, et obtient un brouillon de skills + decisions à valider avant intégration au context cible.

> **Pilier load-bearing #2 (3ème source d'apprentissage).** Un freelance a 5-30 projets passés. C'est une mine que **personne ne valorise** aujourd'hui. Workflow serait le premier à dire "donne-moi ton ancien projet React, je l'absorbe et code comme toi sur les prochains".

## Dépendances

- Phase 1-3 ✅ (`SkillManager`, `TeachSystem`, `ContextRegistry`)
- Phase 9 tâche 9.1 ✅ (`CodeIndexer` — fait l'analyse statique)

## Fichiers à Créer

- `src/workflow/tools/project_ingester.py` [CRÉER]
- `tests/integration/test_project_ingester.py` [CRÉER]

## CLI

```bash
# Forme basique avec opt-in tags
workflow learn-from ~/projects/old-react-app \
  --context web.nextjs \
  --learn architecture,naming,patterns \
  --ignore styling,deps

# Sans tags = explore mode (Workflow propose, dev valide chaque catégorie)
workflow learn-from ~/projects/old-react-app --context web.nextjs

# Dry-run pour voir avant d'absorber
workflow learn-from ~/projects/old-react-app --context web.nextjs --dry-run
```

## Catégories Apprenables

Le dev choisit explicitement (opt-in) ce qu'il veut absorber :

| Tag | Ce qui est extrait |
|-----|-------------------|
| `architecture` | Structure de dossiers, séparation des couches, organisation des modules |
| `naming` | Conventions de nommage (variables, fichiers, exports) |
| `patterns` | Patterns récurrents (custom hooks, util functions, error handling style) |
| `style` | Formatage, indentation, convention de commentaires |
| `decisions` | Décisions techniques inférées depuis README, ADRs, commits |
| `deps` | Choix de dépendances (React Query vs SWR, Zod vs Yup, etc.) |
| `testing` | Style de tests (BDD vs TDD, AAA, mocks vs vrais services) |

Workflow extrait **uniquement** ce que le dev tag. Pas d'absorption silencieuse.

## Pipeline d'Ingestion

```
1. Validation projet
   - Le path existe et contient du code (au moins un fichier source)
   - Détection langue dominante via CodeIndexer

2. Scan structurel (sans LLM)
   - CodeIndexer.rebuild() sur le projet cible (mais en mode read-only, pas commit)
   - Statistiques : N fichiers, M symboles, profondeur, etc.
   - Détection des patterns récurrents (regex + AST)

3. Synthèse LLM (role='curator')
   - Pour chaque catégorie demandée : extraire un brouillon de skills
   - Génère un fichier "ingestion-draft.json" pas encore commit

4. Review utilisateur
   - CLI affiche skill par skill : titre, description, exemple
   - [O] Adopter  [E] Éditer  [N] Skip  [A] Tout adopter restant

5. Commit
   - Skills validés ajoutés au context cible (confidence: HIGH, source: project_ingestion)
   - Decisions inférées vont dans le decisions.log du context cible
   - Audit log dans ~/.workflow/ingestions.log
```

## Implémentation

```python
# src/workflow/tools/project_ingester.py
import json
from pathlib import Path
from datetime import datetime, timezone
from dataclasses import dataclass, field

from workflow.core.context_registry import ContextRegistry
from workflow.core.skill_manager import SkillManager
from workflow.core.decisions_log import DecisionsLog
from workflow.tools.code_indexer import CodeIndexer
from workflow.llm.llm_provider import LLMProvider

CATEGORIES = ['architecture', 'naming', 'patterns', 'style', 'decisions', 'deps', 'testing']


@dataclass
class IngestionProposal:
    category: str
    name: str
    title: str
    description: str
    body: str
    examples: list[str] = field(default_factory=list)


class ProjectIngester:
    def __init__(
        self,
        registry: ContextRegistry,
        skills: SkillManager,
        llm: LLMProvider,
        io,
    ):
        self.registry = registry
        self.skills = skills
        self.llm = llm
        self.io = io

    async def ingest(
        self,
        project_path: Path,
        target_context: str,
        learn: list[str] | None = None,
        ignore: list[str] | None = None,
        dry_run: bool = False,
    ) -> dict:
        """
        Pipeline complet : analyse → synthèse → review → commit.
        """
        if not project_path.exists():
            raise ValueError(f'Projet introuvable : {project_path}')

        ctx = self.registry.get(target_context)
        if not ctx:
            raise ValueError(f'Context cible inexistant : {target_context}')

        categories = self._resolve_categories(learn, ignore)
        self.io.print_header(f'Ingestion — {project_path.name} → {target_context}')
        self.io.print_info(f'Catégories à analyser : {", ".join(categories)}')

        # Étape 1-2 : analyse statique
        analysis = await self._analyze_project(project_path)
        self.io.print(f'  📊 {analysis["files_count"]} fichiers, {analysis["symbols_count"]} symboles')

        # Étape 3 : synthèse LLM par catégorie
        proposals: list[IngestionProposal] = []
        for category in categories:
            self.io.print_info(f'Analyse catégorie : {category}...')
            proposals_for_cat = await self._synthesize_category(
                category, analysis, project_path
            )
            proposals.extend(proposals_for_cat)

        if not proposals:
            self.io.print_warning('Aucun pattern trouvé.')
            return {'adopted': 0, 'skipped': 0}

        if dry_run:
            self.io.print_header(f'Dry-run — {len(proposals)} skills proposés')
            for p in proposals:
                self.io.print(f'  📦 {p.category} :: {p.title}')
                self.io.print(f'     {p.description}')
            return {'proposed': len(proposals), 'adopted': 0}

        # Étape 4 : review utilisateur
        adopted = await self._review_and_commit(proposals, target_context)

        # Étape 5 : audit
        await self._log_ingestion(project_path, target_context, adopted)

        self.io.print_success(
            f'Ingestion terminée : {len(adopted)}/{len(proposals)} skills adoptés.'
        )
        return {'proposed': len(proposals), 'adopted': len(adopted)}

    @staticmethod
    def _resolve_categories(learn: list[str] | None, ignore: list[str] | None) -> list[str]:
        if learn:
            return [c for c in learn if c in CATEGORIES]
        # Sans --learn : tout sauf --ignore
        ignore_set = set(ignore or [])
        return [c for c in CATEGORIES if c not in ignore_set]

    # ─── Analyse statique ─────────────────────────────────────────────

    async def _analyze_project(self, project_path: Path) -> dict:
        """Scan le projet et collecte les stats brutes (sans LLM)"""
        # Indexer en mode "read-only" — on n'écrit pas dans le projet cible
        temp_indexer = CodeIndexer(str(project_path))
        await temp_indexer.rebuild()

        files = temp_indexer._index['files']
        symbols = temp_indexer._index['by_symbol']

        # Detection conventions de nommage
        naming_samples = self._sample_names(files)

        # Détection deps
        deps = self._extract_dependencies(project_path)

        # Lecture README/ADRs
        readme_text = self._read_first_existing([
            project_path / 'README.md',
            project_path / 'README.rst',
        ])
        adr_dir = project_path / 'docs' / 'adr'
        adrs = []
        if adr_dir.exists():
            adrs = [f.read_text(errors='ignore')[:1500] for f in adr_dir.glob('*.md')][:10]

        return {
            'project_path': str(project_path),
            'files_count': len(files),
            'symbols_count': len(symbols),
            'languages': self._dominant_languages(files),
            'naming_samples': naming_samples,
            'dependencies': deps,
            'readme_text': readme_text[:3000] if readme_text else '',
            'adrs': adrs,
            'top_symbols': sorted(
                symbols.items(), key=lambda x: len(x[1]), reverse=True
            )[:50],
            'folder_structure': self._top_level_folders(project_path),
        }

    @staticmethod
    def _dominant_languages(files: dict) -> list[str]:
        from collections import Counter
        langs = Counter(f.get('language', 'unknown') for f in files.values())
        return [lang for lang, _ in langs.most_common(3)]

    @staticmethod
    def _sample_names(files: dict, max_per_file: int = 3) -> list[str]:
        samples: list[str] = []
        for file_data in list(files.values())[:30]:
            for sym in file_data.get('symbols', [])[:max_per_file]:
                samples.append(sym['name'])
        return samples[:100]

    @staticmethod
    def _extract_dependencies(project_path: Path) -> dict:
        deps: dict = {}
        for fname in ('package.json', 'pyproject.toml', 'pubspec.yaml', 'Cargo.toml', 'go.mod'):
            f = project_path / fname
            if f.exists():
                deps[fname] = f.read_text(errors='ignore')[:2000]
        return deps

    @staticmethod
    def _top_level_folders(project_path: Path) -> list[str]:
        result: list[str] = []
        for item in sorted(project_path.iterdir()):
            if item.is_dir() and not item.name.startswith('.') and item.name not in (
                'node_modules', 'venv', 'dist', 'build', '__pycache__'
            ):
                result.append(item.name)
        return result[:30]

    @staticmethod
    def _read_first_existing(paths: list[Path]) -> str | None:
        for p in paths:
            if p.exists():
                return p.read_text(encoding='utf-8', errors='ignore')
        return None

    # ─── Synthèse LLM par catégorie ───────────────────────────────────

    async def _synthesize_category(
        self, category: str, analysis: dict, project_path: Path
    ) -> list[IngestionProposal]:
        """Demander au LLM (role='curator') de proposer des skills pour cette catégorie"""
        prompt = self._build_category_prompt(category, analysis)
        try:
            response = await self.llm.ask(prompt, role='curator', max_tokens=4000)
            data = self._extract_json(response)
            proposals_raw = json.loads(data)
        except (json.JSONDecodeError, ValueError) as e:
            self.io.print_warning(f'  ⚠️ Synthèse {category} échouée : {e}')
            return []

        return [
            IngestionProposal(
                category=category,
                name=p.get('name', ''),
                title=p.get('title', p.get('name', '')),
                description=p.get('description', ''),
                body=p.get('body', ''),
                examples=p.get('examples', []),
            )
            for p in proposals_raw
        ]

    def _build_category_prompt(self, category: str, analysis: dict) -> str:
        category_focus = {
            'architecture': 'la structure de dossiers, la séparation des couches, comment les modules sont organisés',
            'naming': 'les conventions de nommage (camelCase vs snake_case, préfixes, suffixes, exports nommés)',
            'patterns': 'les patterns récurrents (custom hooks, error handling, factories, decorators)',
            'style': 'le style de code (formatage, indentation, longueur de lignes, commentaires)',
            'decisions': 'les décisions techniques exprimées dans README/ADRs (choix de libs, principes)',
            'deps': 'les dépendances utilisées et leur rôle (eg "React Query pour caching server state")',
            'testing': 'le style de tests (BDD/TDD, mocks, structure des fichiers de tests)',
        }

        return f"""Tu es le Curator de Workflow. Analyse ce projet existant et extrait les
patterns de la catégorie : **{category}** ({category_focus.get(category, "")}).

Contexte projet :
- Path : {analysis['project_path']}
- Langues : {", ".join(analysis['languages'])}
- {analysis['files_count']} fichiers, {analysis['symbols_count']} symboles
- Folders top-level : {", ".join(analysis['folder_structure'])}
- Échantillon de noms : {", ".join(analysis['naming_samples'][:30])}
- Dépendances : {json.dumps(list(analysis['dependencies'].keys()))}

README extrait :
{analysis['readme_text'][:1500]}

ADRs (s'il y en a) :
{chr(10).join(analysis['adrs'][:3])[:2000]}

Top symbols (les plus définis) :
{json.dumps([s[0] for s in analysis['top_symbols'][:30]])}

Identifie 2 à 5 patterns RÉUTILISABLES dans la catégorie {category}.
Filtre out les choses spécifiques à ce projet (eg "le composant ProductCard a 3 props" — pas réutilisable).
Garde ce qui transfère à un futur projet du même style (eg "components UI ont toujours un fichier .stories.tsx à côté").

Retourne UNIQUEMENT un JSON :
[
  {{
    "name": "slug-court-pour-le-skill",
    "title": "Titre lisible",
    "description": "1 ligne",
    "body": "Description complète markdown du pattern, avec quand l'appliquer",
    "examples": ["exemple 1", "exemple 2"]
  }}
]"""

    @staticmethod
    def _extract_json(text: str) -> str:
        text = text.strip()
        if text.startswith('```'):
            lines = text.split('\n')
            text = '\n'.join(lines[1:-1])
        return text

    # ─── Review interactif ────────────────────────────────────────────

    async def _review_and_commit(
        self, proposals: list[IngestionProposal], target_context: str
    ) -> list[IngestionProposal]:
        """Itérer sur chaque proposal, demander adoption/skip/édition"""
        adopted: list[IngestionProposal] = []
        adopt_all_remaining = False

        for i, p in enumerate(proposals, 1):
            self.io.print_header(f'[{i}/{len(proposals)}] {p.category} :: {p.title}')
            self.io.print(p.description)
            self.io.print('')
            self.io.print(p.body[:400])

            if adopt_all_remaining:
                choice = 'o'
            else:
                choice = self.io.prompt(
                    '[O]adopter / [E]éditer body / [N]skip / [A]tout adopter le reste',
                    default='o',
                ).strip().lower()

            if choice in ('o', 'oui', 'y', 'yes', ''):
                self._commit_skill(p, target_context)
                adopted.append(p)
            elif choice == 'a':
                self._commit_skill(p, target_context)
                adopted.append(p)
                adopt_all_remaining = True
            elif choice == 'e':
                # Éditer dans $EDITOR puis valider
                edited_body = self.io.prompt('Éditer body (ou laisser tel quel)', default=p.body)
                p.body = edited_body
                self._commit_skill(p, target_context)
                adopted.append(p)
            # n / skip → ne rien faire

        return adopted

    def _commit_skill(self, proposal: IngestionProposal, target_context: str):
        skill_data = {
            'name': f'ingested-{proposal.name}',
            'description': proposal.description,
            'tags': ['project_ingestion', proposal.category],
            'body': proposal.body,
            'source': 'project_ingestion',
            'confidence': 'HIGH',
            'context': target_context,
        }
        # Réutilise la mécanique d'écriture de SkillManager / TeachSystem
        ctx = self.registry.get(target_context)
        skill_dir = ctx.path / 'skills'
        skill_dir.mkdir(exist_ok=True)
        self._write_skill(skill_dir, skill_data)

    @staticmethod
    def _write_skill(skill_dir: Path, skill_data: dict):
        import yaml
        path = skill_dir / f'{skill_data["name"]}.md'
        fm = {
            'name': skill_data['name'],
            'description': skill_data['description'],
            'version': '1.0',
            'source': skill_data['source'],
            'confidence': skill_data['confidence'],
            'context': skill_data['context'],
            'tags': skill_data.get('tags', []),
            'usage_count': 0,
            'last_used': None,
            'created_at': datetime.now(timezone.utc).date().isoformat(),
        }
        content = (
            '---\n'
            + yaml.dump(fm, allow_unicode=True, default_flow_style=False)
            + '---\n\n'
            + skill_data['body']
        )
        path.write_text(content, encoding='utf-8')

    # ─── Audit ────────────────────────────────────────────────────────

    async def _log_ingestion(
        self, project_path: Path, target_context: str, adopted: list[IngestionProposal]
    ):
        """Écrire un audit log dans ~/.workflow/ingestions.log"""
        log = Path.home() / '.workflow' / 'ingestions.log'
        log.parent.mkdir(parents=True, exist_ok=True)
        entry = (
            f"[{datetime.now(timezone.utc).isoformat()}] "
            f"FROM={project_path} TO_CONTEXT={target_context} "
            f"SKILLS_ADOPTED={len(adopted)}\n"
        )
        with open(log, 'a', encoding='utf-8') as f:
            f.write(entry)
            for p in adopted:
                f.write(f"  + {p.category}/{p.name}\n")
```

## Intégration

### CLI (Phase 5 — extension)

```python
@app.command()
def learn_from(
    path: Path,
    context: str = typer.Option(..., '--context', '-c'),
    learn: list[str] = typer.Option([], '--learn'),
    ignore: list[str] = typer.Option([], '--ignore'),
    dry_run: bool = typer.Option(False, '--dry-run'),
):
    asyncio.run(ingester.ingest(
        path,
        target_context=context,
        learn=learn or None,
        ignore=ignore or None,
        dry_run=dry_run,
    ))
```

### MCP (Phase 6 — outil)

```
workflow_learn_from(project_path, target_context, learn=[], ignore=[], dry_run=false)
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `ingest()` valide que le projet existe et le context cible existe | ⬜ |
| 2 | `_resolve_categories()` respecte `--learn` (whitelist) et `--ignore` (blacklist) | ⬜ |
| 3 | `_analyze_project()` ne touche PAS au projet source (read-only) | ⬜ |
| 4 | `_analyze_project()` extrait stats : files_count, symbols_count, languages, naming_samples | ⬜ |
| 5 | `_synthesize_category('naming', ...)` génère 2-5 proposals via LLM curator | ⬜ |
| 6 | `dry_run=True` n'écrit aucun skill, retourne juste les proposals | ⬜ |
| 7 | Le review interactif supporte O/E/N/A (adopt/edit/skip/all) | ⬜ |
| 8 | Skills créés ont `source='project_ingestion'`, `confidence='HIGH'`, tag de catégorie | ⬜ |
| 9 | `ingestions.log` enregistre chaque ingestion avec timestamp + path + adopted | ⬜ |
| 10 | Tests : 5 catégories, mock LLM, dry-run, review path adopt-all | ⬜ |

## Notes d'Implémentation

- **Pas d'absorption silencieuse** : chaque skill est validé un par un. C'est lent mais essentiel — le dev doit garder le contrôle. Le mode `--auto-adopt` n'existe PAS.
- **Quotas LLM** : ingester un gros projet (>500 fichiers) coûte cher en tokens. Limiter l'analyse à 30-50 fichiers représentatifs (top symboles).
- **Privacy** : le projet source peut contenir des secrets — exclure les fichiers `.env`, `secrets/`, etc. avant analyse.
- **Reproductibilité** : si le dev relance `learn-from` sur le même projet, deduplication via skill name (`ingested-<slug>` → skip si existe).
- **Cross-projet** : si le dev ingest 3 projets web Next.js dans `web.nextjs`, le Curator détectera après les patterns convergents et pourra promouvoir au parent `web`.
