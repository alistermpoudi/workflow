# Phase 6 — Tâche 6.5 : BreakingChangeDetector.py

## Objectif

Avant de commencer à coder une tâche, analyser les impacts de cette modification sur le reste du codebase. Si des fichiers touchés par la tâche sont importés par d'autres modules — surtout des modules liés à des tâches déjà DONE — Workflow avertit le développeur et propose de générer des tests de régression **avant** de modifier quoi que ce soit.

> **Le principe** : les régressions sont 10x moins chères à prévenir qu'à déboguer. `BreakingChangeDetector` transforme une erreur découverte en prod en une vérification automatique pré-codage.

## Dépendances

- Phase 1 (`FileSystem`, `TaskManager`) ✅
- Tâche 6.2 (`CodeIndexer`) ✅ — pour l'index des imports

## Fichiers à Créer

- `src/workflow/tools/breaking_change_detector.py` [CRÉER]

## Concept : Graphe d'Impact

```
TASK-012 modifie : src/auth/middleware.py, src/auth/jwt_service.py

CodeIndexer révèle que ces fichiers sont importés par :
  src/routes/dashboard.py       ← lié à TASK-004 (DONE ✅)
  src/routes/clients.py         ← lié à TASK-007 (DONE ✅)
  src/routes/invoices.py        ← lié à TASK-009 (DONE ✅)
  tests/test_auth.py            ← lié à TASK-002 (DONE ✅)

BreakingChangeDetector :
  "⚠ TASK-012 touche 4 modules de tâches déjà terminées.
   Veux-tu générer des tests de régression avant de coder ?"
```

## Implémentation

```python
# src/workflow/tools/breaking_change_detector.py
import asyncio
import re
from pathlib import Path

from workflow.core.project_memory import ProjectMemory
from workflow.tools.task_manager import TaskManager
from workflow.llm.llm_provider import LLMProvider

REGRESSION_TEST_PROMPT = """Génère des tests de régression pour vérifier que ces modules continuent de fonctionner correctement.

Fichiers à risque (importent un module qui va être modifié) :
{at_risk_files}

Contenu actuel des fichiers :
{file_contents}

Génère des tests qui vérifient le comportement ACTUEL de ces modules.
Ces tests doivent passer AVANT la modification et doivent continuer à passer APRÈS.

Format : ### tests/regression/{module_name}_regression.py suivi du code."""


class BreakingChangeDetector:
    def __init__(self, project_root: str, llm: LLMProvider):
        self.project_root = Path(project_root).resolve()
        self.memory = ProjectMemory(project_root)
        self.tasks = TaskManager(project_root)
        self.llm = llm

    async def analyze(self, version: str, task: dict) -> dict:
        """
        Analyser l'impact d'une tâche sur le codebase existant.
        Retourne un rapport d'impact avec les fichiers à risque.
        """
        files_to_modify = [
            f.get('path', str(f)) if isinstance(f, dict) else str(f)
            for f in (task.get('files') or task.get('files_to_modify') or [])
        ]

        if not files_to_modify:
            return {'impact': 'none', 'at_risk': []}

        # Construire l'index des imports depuis le codebase
        import_graph = await self._build_import_graph(files_to_modify)
        if not import_graph:
            return {'impact': 'none', 'at_risk': []}

        # Trouver les tâches DONE dont les fichiers importent les modules touchés
        done_tasks = await self._get_done_tasks_impact(version, import_graph)

        impact_level = (
            'critical' if len(done_tasks) >= 3 else
            'high' if len(done_tasks) >= 1 else
            'none'
        )

        return {
            'impact': impact_level,
            'files_to_modify': files_to_modify,
            'at_risk_files': list(import_graph.keys()),
            'affected_done_tasks': done_tasks,
            'total_at_risk': len(import_graph),
        }

    async def generate_regression_tests(self, at_risk_files: list[str]) -> str:
        """Générer des tests de régression pour les fichiers à risque."""
        contents = {}
        for file_path in at_risk_files[:5]:  # Limiter à 5 fichiers
            full_path = self.project_root / file_path
            if full_path.exists():
                contents[file_path] = full_path.read_text(encoding='utf-8')[:800]

        if not contents:
            return ''

        file_contents = '\n\n'.join(
            f'### {path}\n```python\n{content}\n```'
            for path, content in contents.items()
        )

        return await self.llm.ask(
            REGRESSION_TEST_PROMPT.format(
                at_risk_files='\n'.join(f'- {f}' for f in at_risk_files),
                file_contents=file_contents,
            ),
            role='code_generation',
        )

    async def _build_import_graph(self, files_to_modify: list[str]) -> dict[str, list[str]]:
        """
        Pour chaque fichier à modifier, trouver les fichiers du projet qui l'importent.
        Utilise ripgrep pour la recherche — rapide sur les grands codebases.
        """
        graph: dict[str, list[str]] = {}

        for target_file in files_to_modify:
            # Dériver le nom du module depuis le chemin
            module_name = self._path_to_module(target_file)
            if not module_name:
                continue

            # Chercher les imports avec ripgrep
            importers = await self._find_importers(module_name, target_file)
            if importers:
                graph[target_file] = importers

        return graph

    async def _find_importers(self, module_name: str, file_path: str) -> list[str]:
        """Trouver les fichiers qui importent ce module via ripgrep."""
        patterns = [
            f'import {module_name}',
            f'from {module_name}',
            f'from .{module_name.split(".")[-1]}',
        ]

        results = set()
        for pattern in patterns:
            proc = await asyncio.create_subprocess_exec(
                'rg', '--type', 'py', '-l', pattern, str(self.project_root),
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.DEVNULL,
            )
            stdout, _ = await proc.communicate()
            if proc.returncode == 0:
                for line in stdout.decode('utf-8').splitlines():
                    rel = str(Path(line).relative_to(self.project_root))
                    if rel != file_path and not rel.startswith('.workflow'):
                        results.add(rel)

        return list(results)

    async def _get_done_tasks_impact(
        self, version: str, import_graph: dict[str, list[str]]
    ) -> list[dict]:
        """
        Identifier les tâches DONE dont les fichiers sont dans le graphe d'impact.
        """
        progress = await self.memory.get_progress(version)
        done_ids = progress.get('done', []) if progress else []

        at_risk_files = set()
        for importers in import_graph.values():
            at_risk_files.update(importers)

        impacted = []
        for task_id in done_ids:
            task = await self.tasks.get_task(version, task_id)
            if not task:
                continue
            task_files = {
                f.get('path', '') if isinstance(f, dict) else str(f)
                for f in (task.get('files') or [])
            }
            overlap = at_risk_files & task_files
            if overlap:
                impacted.append({
                    'task_id': task_id,
                    'title': task.get('title', ''),
                    'at_risk_files': list(overlap),
                })

        return impacted

    def _path_to_module(self, file_path: str) -> str:
        """Convertir un chemin de fichier en nom de module Python."""
        path = Path(file_path)
        if path.suffix != '.py':
            return ''
        # src/workflow/auth/jwt_service.py → workflow.auth.jwt_service
        parts = path.with_suffix('').parts
        # Supprimer 'src' si présent en tête
        if parts and parts[0] == 'src':
            parts = parts[1:]
        return '.'.join(parts)
```

## Intégration dans `ExecutionPhase`

```python
# Dans ExecutionPhase.run(), avant de déléguer à ExecutionLoop

from workflow.tools.breaking_change_detector import BreakingChangeDetector

detector = BreakingChangeDetector(self.project_root, self.llm)
impact = await detector.analyze(version, task)

if impact['impact'] in ('critical', 'high'):
    self.io.print_warning(
        f'[Impact] {task_id} touche {impact["total_at_risk"]} module(s) '
        f'utilisé(s) par {len(impact["affected_done_tasks"])} tâche(s) déjà terminée(s) :'
    )
    for affected in impact['affected_done_tasks']:
        self.io.print(
            f'  - {affected["task_id"]} ({affected["title"]}) '
            f'→ {", ".join(affected["at_risk_files"])}'
        )

    generate_tests = self.io.confirm(
        'Générer des tests de régression avant de coder ?', default=True
    )
    if generate_tests:
        self.io.print_info('Génération des tests de régression...')
        regression_code = await detector.generate_regression_tests(impact['at_risk_files'])
        if regression_code:
            # Appliquer les tests de régression dans le worktree courant
            from workflow.tools.execution_loop import ExecutionLoop
            temp_loop = ExecutionLoop(self.project_root, self.llm, self.io, tech_stack)
            temp_loop._apply_generated_files(regression_code)
            reg_result = await temp_loop._run_command(tech_stack.get('test', ''))
            if reg_result['exit_code'] == 0:
                self.io.print_success('Tests de régression générés et passants ✓')
            else:
                self.io.print_warning('Tests de régression générés mais certains échouent déjà.')
```

## Rapport d'Impact — Exemple CLI

```
workflow run

[Impact] TASK-012 (Auth middleware refactor) touche 3 module(s) utilisé(s) par 4 tâche(s) terminées :

  - TASK-004 (Dashboard) → src/routes/dashboard.py
  - TASK-007 (Clients CRUD) → src/routes/clients.py
  - TASK-009 (Invoices) → src/routes/invoices.py

Générer des tests de régression avant de coder ? [O/n] O

Génération des tests de régression...
✓ Tests de régression générés et passants (12 tests)

[Code] Génération de TASK-012...
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `_find_importers()` utilise `ripgrep` via subprocess | ⬜ |
| 2 | `_path_to_module()` convertit `src/workflow/auth/jwt.py` → `workflow.auth.jwt` | ⬜ |
| 3 | `analyze()` retourne `impact: 'none'` si aucun importer trouvé | ⬜ |
| 4 | `analyze()` retourne `impact: 'critical'` si 3+ tâches DONE affectées | ⬜ |
| 5 | `generate_regression_tests()` utilise `role='code_generation'` | ⬜ |
| 6 | L'intégration dans `ExecutionPhase` affiche la liste des tâches affectées | ⬜ |
| 7 | L'utilisateur peut refuser la génération des tests de régression | ⬜ |
| 8 | Les fichiers `.workflow/` sont exclus du graphe d'impact | ⬜ |
| 9 | `_get_done_tasks_impact()` vérifie l'overlap entre at_risk_files et task.files | ⬜ |
