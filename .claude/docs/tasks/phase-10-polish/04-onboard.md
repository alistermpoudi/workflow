# Phase 10 — Tâche 10.4 : workflow onboard

## Objectif

Créer la commande `workflow onboard` — l'**onboarding instantané** d'un nouveau développeur sur un projet Workflow existant. En 30 secondes, le nouveau dev comprend : la vision, la stack, l'état d'avancement, les 5 décisions clés, la prochaine tâche suggérée, et les conventions de l'équipe. Lit toute la `.workflow/` et synthétise via LLM.

> **Le test ultime de la promesse "mémoire institutionnelle"** : si onboard ne donne pas une compréhension réelle en 30 secondes, le pilier #2 ne tient pas.

## Dépendances

- Phases 1-6 ✅
- Phase 9 ✅ (`CodeIndexer` pour les stats)

## Fichiers à Créer

- `src/workflow/core/onboarding_manager.py` [CRÉER]
- Outil MCP `workflow_onboard()` (déjà déclaré, à implémenter ici)
- Commande CLI `workflow onboard`

## Sortie Attendue

```
═══════════════════════════════════════════════════════════════
  ONBOARDING — TaskFlow
═══════════════════════════════════════════════════════════════

📂 Projet
  Application de gestion de tâches collaborative.
  Cible : équipes freelance distribuées (3-15 personnes).

🛠 Stack
  Python 3.12 / FastAPI / PostgreSQL / Prisma
  Tests : pytest    Lint : ruff
  Build : uv run ruff check . && uv run mypy src/

📊 Avancement
  Phase actuelle : VALIDATION
  Version active : v1.5 (3/12 tâches done)
  v1.0 COMPLETED — 24 tâches livrées

🧠 5 Décisions Clés à Connaître
  1. ORM : Prisma (12/03 — DX + migrations fiables sur PG)
  2. Auth : JWT uniquement (18/03 — serverless requis)
  3. Caching : Redis (02/04 — sessions + rate limit)
  4. API : REST puis GraphQL en v2 (12/04 — DX d'abord)
  5. Tests : pytest + factories Boy (15/04 — équipe Python-only)

📌 Conventions Équipe
  • Pas de stash automatique — commiter ou stash manuel
  • allowed_commands est versionnée — modifier via `/allow` (Telegram) ou MCP
  • Chaque PR = 1 tâche atomique avec ses tests

🎯 Prochaine Tâche Suggérée
  TASK-027 : "Endpoint POST /tasks/bulk-import"
  Fichiers : src/api/tasks.py, src/services/import.py
  Dépendances : ✅ TASK-024, ✅ TASK-025
  Estimation : ~3h (médiane projet)

🔥 Skills Réutilisables Détectés
  • prisma-migration (utilisé 8 fois)
  • jwt-refresh-pattern (utilisé 5 fois)
  • pytest-async-fixtures (utilisé 12 fois)

⚠️  Points d'attention
  • 1 contradiction de décision détectée : DEC-018 vs DEC-022 (cache strategy)
  • TASK-019 reportée 2x — escalation recommandée

═══════════════════════════════════════════════════════════════
Lance `workflow run` pour démarrer la prochaine tâche.
```

## Implémentation

```python
# src/workflow/core/onboarding_manager.py
import asyncio
from pathlib import Path

from workflow.core.project_memory import ProjectMemory
from workflow.core.decisions_log import DecisionsLog
from workflow.core.decisions_graph import DecisionsGraph
from workflow.core.skill_manager import SkillManager
from workflow.core.context_manager import ContextManager
from workflow.tools.task_manager import TaskManager
from workflow.tools.code_indexer import CodeIndexer
from workflow.llm.llm_provider import LLMProvider


class OnboardingManager:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.memory = ProjectMemory(project_root)
        self.decisions = DecisionsLog(project_root)
        self.graph = DecisionsGraph(project_root, llm)
        self.skills = SkillManager(project_root)
        self.tasks = TaskManager(project_root)
        self.indexer = CodeIndexer(project_root, llm)
        self.context = ContextManager(project_root, llm)
        self.llm = llm
        self.io = io

    async def generate(self) -> dict:
        """Générer le briefing d'onboarding complet"""
        project = await self.memory.get_project()
        if not project:
            return {'error': 'Aucun projet .workflow/ trouvé'}

        # Lecture parallèle des artéfacts
        vision_md, features, tech_stack, design = await asyncio.gather(
            self.memory.get_vision(),
            self.memory.get_features(),
            self.memory.get_tech_stack(),
            self.memory.get_design(),
        )

        sections = await asyncio.gather(
            self._project_summary(project, vision_md),
            self._stack_summary(tech_stack),
            self._progress_summary(project),
            self._key_decisions(),
            self._team_conventions(tech_stack),
            self._next_task(project),
            self._top_skills(),
            self._warnings(),
        )

        return {
            'project': sections[0],
            'stack': sections[1],
            'progress': sections[2],
            'key_decisions': sections[3],
            'conventions': sections[4],
            'next_task': sections[5],
            'top_skills': sections[6],
            'warnings': sections[7],
        }

    # ─── Sous-générateurs ─────────────────────────────────────────────

    async def _project_summary(self, project: dict, vision_md: str | None) -> dict:
        """Résumé court de la vision via LLM"""
        if not vision_md:
            return {
                'name': project.get('name', '?'),
                'summary': project.get('description', ''),
            }
        prompt = f"""Résume cette vision produit en 2 phrases courtes
(application + cible utilisateur). Pas de jargon.

{vision_md[:2000]}"""
        summary = await self.llm.ask(prompt, role='fast', max_tokens=200)
        return {
            'name': project.get('name', '?'),
            'summary': summary.strip(),
        }

    async def _stack_summary(self, tech_stack: dict | None) -> dict:
        if not tech_stack:
            return {}
        return {
            'language': tech_stack.get('language'),
            'framework': tech_stack.get('framework'),
            'database': tech_stack.get('database'),
            'orm': tech_stack.get('orm'),
            'test_command': tech_stack.get('test'),
            'build_command': tech_stack.get('build_validate'),
        }

    async def _progress_summary(self, project: dict) -> dict:
        active_version = project.get('active_version')
        if not active_version:
            return {'phase': project.get('status'), 'versions': []}

        progress = await self.memory.get_progress(active_version)
        all_versions = []
        # Lister toutes les versions et leur statut
        versions_dir = Path(self.memory.project_root) / '.workflow' / 'versions'
        if versions_dir.exists():
            for v_dir in sorted(versions_dir.iterdir()):
                if v_dir.is_dir():
                    meta = await self.memory.get_version_meta(v_dir.name)
                    if meta:
                        v_progress = await self.memory.get_progress(v_dir.name) or {}
                        all_versions.append({
                            'name': v_dir.name,
                            'status': meta.get('status'),
                            'done': len(v_progress.get('done', [])),
                            'pending': len(v_progress.get('pending', [])),
                        })

        return {
            'phase': project.get('status'),
            'active_version': active_version,
            'active_done': len(progress.get('done', [])),
            'active_pending': len(progress.get('pending', [])),
            'all_versions': all_versions,
        }

    async def _key_decisions(self, max_items: int = 5) -> list[dict]:
        """5 décisions HIGH-confidence les plus impactantes"""
        all_decisions = await self.decisions.get_all()
        high = [d for d in all_decisions if d.get('confidence') == 'HIGH']
        # Tri par récence + nombre de tâches qui les référencent
        high.sort(
            key=lambda d: (d.get('date', ''), -len(d.get('referenced_by', []))),
            reverse=True,
        )
        return [
            {
                'id': d.get('id'),
                'date': d.get('date'),
                'summary': d.get('summary'),
                'reason': d.get('reason'),
            }
            for d in high[:max_items]
        ]

    async def _team_conventions(self, tech_stack: dict | None) -> list[str]:
        """Lister les conventions principales du projet"""
        conventions = [
            'Pas de stash automatique — commiter ou stash manuel avant version switch',
            '1 PR = 1 tâche atomique mergeable avec ses tests',
        ]
        if tech_stack and tech_stack.get('allowed_commands'):
            conventions.append(
                'allowed_commands est partagée — modifier via `/allow` (Telegram) ou MCP'
            )
        return conventions

    async def _next_task(self, project: dict) -> dict | None:
        """Prochaine tâche prête (dépendances satisfaites)"""
        active_version = project.get('active_version')
        if not active_version:
            return None
        progress = await self.memory.get_progress(active_version) or {}
        pending = progress.get('pending', [])
        done = set(progress.get('done', []))

        for task_id in pending:
            task = await self.tasks.get_task(active_version, task_id)
            if not task:
                continue
            deps = set(task.get('dependencies') or [])
            if deps.issubset(done):
                return {
                    'id': task_id,
                    'title': task.get('title'),
                    'files': [f['path'] for f in task.get('files') or []],
                    'dependencies': list(deps),
                }
        return None

    async def _top_skills(self, max_items: int = 5) -> list[dict]:
        """5 skills les plus utilisés sur ce projet"""
        all_skills = self.skills.list_all()
        all_skills.sort(key=lambda s: s.get('usage_count', 0), reverse=True)
        return [
            {
                'name': s.get('name'),
                'usage_count': s.get('usage_count', 0),
                'description': s.get('description'),
            }
            for s in all_skills[:max_items]
            if s.get('usage_count', 0) > 0
        ]

    async def _warnings(self) -> list[str]:
        """Détecter les points d'attention pour le nouveau dev"""
        warnings = []

        # Contradictions actives
        contradictions = await self.graph.find_active_contradictions()
        if contradictions:
            warnings.append(
                f'{len(contradictions)} contradiction(s) de décisions actives — voir decisions-graph.json'
            )

        # Tâches échouées
        project = await self.memory.get_project()
        active_version = (project or {}).get('active_version')
        if active_version:
            progress = await self.memory.get_progress(active_version) or {}
            failed = progress.get('failed', [])
            for task_id in failed:
                task = await self.tasks.get_task(active_version, task_id)
                if task and len(task.get('journal', [])) >= 2:
                    warnings.append(
                        f'{task_id} reportée {len(task["journal"])}× — escalation recommandée'
                    )

        return warnings

    # ─── Rendu ────────────────────────────────────────────────────────

    def render(self, briefing: dict) -> str:
        """Rendu texte du briefing pour CLI"""
        sep = '═' * 63
        lines = [
            sep,
            f"  ONBOARDING — {briefing['project']['name']}",
            sep,
            '',
            '📂 Projet',
            f"  {briefing['project'].get('summary', '—')}",
            '',
            '🛠 Stack',
        ]
        s = briefing.get('stack', {})
        if s:
            stack_parts = [s.get(k) for k in ('language', 'framework', 'database', 'orm') if s.get(k)]
            lines.append(f"  {' / '.join(stack_parts)}")
            lines.append(f"  Tests : {s.get('test_command', '—')}")
            lines.append(f"  Build : {s.get('build_command', '—')}")
        lines.append('')

        p = briefing.get('progress', {})
        lines.append('📊 Avancement')
        lines.append(f"  Phase : {p.get('phase', '—')}")
        if p.get('active_version'):
            lines.append(
                f"  Version active : {p['active_version']} "
                f"({p.get('active_done', 0)}/{p.get('active_done', 0) + p.get('active_pending', 0)} tâches)"
            )
        for v in p.get('all_versions', []):
            if v['status'] == 'COMPLETED':
                lines.append(f"  {v['name']} COMPLETED — {v['done']} tâches livrées")
        lines.append('')

        lines.append('🧠 5 Décisions Clés à Connaître')
        for i, d in enumerate(briefing.get('key_decisions', []), 1):
            lines.append(f"  {i}. {d.get('summary', '')} ({d.get('date', '')})")
            lines.append(f"     {d.get('reason', '')}")
        lines.append('')

        lines.append('📌 Conventions Équipe')
        for c in briefing.get('conventions', []):
            lines.append(f"  • {c}")
        lines.append('')

        nt = briefing.get('next_task')
        if nt:
            lines.append('🎯 Prochaine Tâche Suggérée')
            lines.append(f"  {nt['id']} : {nt.get('title', '')}")
            lines.append(f"  Fichiers : {', '.join(nt.get('files', []))}")
            lines.append('')

        skills = briefing.get('top_skills', [])
        if skills:
            lines.append('🔥 Skills Réutilisables')
            for s in skills:
                lines.append(f"  • {s['name']} (utilisé {s['usage_count']} fois)")
            lines.append('')

        warnings = briefing.get('warnings', [])
        if warnings:
            lines.append('⚠️  Points d\'attention')
            for w in warnings:
                lines.append(f"  • {w}")
            lines.append('')

        lines.append(sep)
        lines.append('Lance `workflow run` pour démarrer la prochaine tâche.')
        return '\n'.join(lines)
```

## Intégration

**CLI (Phase 5)** :
```python
@app.command()
def onboard():
    """Onboarding instantané d'un nouveau dev en 30 secondes"""
    agent = WorkflowAgent('.', io)
    asyncio.run(agent.onboard_command())
```

**MCP Server (Phase 6)** — outil `workflow_onboard` retourne le `briefing` JSON brut, pour qu'un client (VS Code, web) puisse le re-rendre.

**Telegram Bot (Phase 8)** — `/onboard` envoie le rendu Markdown.

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `generate()` lit toutes les sources `.workflow/` en parallèle (asyncio.gather) | ⬜ |
| 2 | `_project_summary()` génère un résumé en 2 phrases via LLM `role='fast'` | ⬜ |
| 3 | `_key_decisions()` retourne 5 décisions HIGH-confidence triées par récence + impact | ⬜ |
| 4 | `_next_task()` respecte les dépendances (ne propose pas une tâche bloquée) | ⬜ |
| 5 | `_warnings()` détecte les contradictions actives via `DecisionsGraph` | ⬜ |
| 6 | `_warnings()` détecte les tâches reportées 2× | ⬜ |
| 7 | `_top_skills()` ne liste que les skills `usage_count > 0` | ⬜ |
| 8 | `render()` produit un rendu texte lisible (CLI) | ⬜ |
| 9 | L'ensemble se génère en < 5s sur un projet de 24 tâches (sans cache LLM) | ⬜ |
| 10 | Tests : projet vide (gère gracieusement), projet riche (rendu complet) | ⬜ |

## Notes d'Implémentation

- **Cache LLM** : le résumé de vision est stable — cacher le résultat dans `.workflow/onboarding-cache.json` invalidé par hash de `vision.md`.
- **Format JSON pour MCP** : retourner `briefing` sans rendre, laisser le client choisir le rendu (texte, JSON, HTML, Markdown).
- **Détection langue** : si la vision est en anglais, basculer le ton du LLM en anglais aussi.
- **Workflow team mode** : si plusieurs `approved_by` dans `allowed-commands.json`, ajouter une section "Équipe" avec les contributeurs.
