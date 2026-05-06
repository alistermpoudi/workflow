# Phase 2 — Tâche 2.6 : ExecutionPhase.py

## Objectif

Créer `ExecutionPhase.py` — l'orchestrateur de la phase d'exécution. Il sélectionne la prochaine tâche `pending`, charge le contexte approprié via `ContextManager`, délègue à `ExecutionLoop` (Phase 3), et met à jour `progress.json`.

> `ExecutionPhase` est le **chef d'orchestre** — il ne code pas lui-même. Il sait quelle tâche
> prendre, dans quel ordre, et délègue le travail réel à `ExecutionLoop`.

## Dépendances

- Tâches 2.1 ✅, 2.2 ✅, 2.3 ✅
- Phase 1 ✅
- `ExecutionLoop` (Phase 3 — tâche 3.1) sera câblé plus tard

## Fichiers à Créer

- `src/workflow/phases/execution_phase.py` [CRÉER]

## Implémentation

```python
# src/workflow/phases/execution_phase.py
from workflow.core.project_memory import ProjectMemory
from workflow.core.context_manager import ContextManager
from workflow.tools.task_manager import TaskManager
from workflow.llm.llm_provider import LLMProvider


class ExecutionPhase:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.memory = ProjectMemory(project_root)
        self.context = ContextManager(project_root, llm)
        self.tasks = TaskManager(project_root)
        self.llm = llm
        self.io = io
        self._execution_loop = None  # Injecté par WorkflowAgent (Phase 3)

    def set_execution_loop(self, loop) -> None:
        """Injecter ExecutionLoop depuis WorkflowAgent (Phase 3)"""
        self._execution_loop = loop

    async def run(self) -> dict:
        """
        Sélectionner et exécuter la prochaine tâche pending.
        Retourne {'completed': True} quand toutes les tâches de la version sont done.
        """
        project = await self.memory.get_project()
        version = (project or {}).get('active_version')

        if not version:
            self.io.print_error('Aucune version active — crée une version d\'abord.')
            return {'completed': False}

        # Charger le contexte version (niveau 2 — IDs seulement)
        version_ctx = await self.context.get_version_context(version)

        pending = version_ctx.get('pending_tasks', [])
        if not pending:
            done = version_ctx.get('done_tasks', [])
            self.io.print_success(f'Toutes les tâches de {version} sont terminées ({len(done)} tâches).')
            return {'completed': True}

        # Vérifier les prérequis de la prochaine tâche
        task_id = await self._next_ready_task(version, pending, version_ctx.get('done_tasks', []))
        if not task_id:
            self.io.print_warning('Aucune tâche prête — des dépendances sont en attente.')
            return {'completed': False}

        self.io.print_header(f'Exécution — {task_id}')

        # Charger le contexte tâche complet (niveau 3)
        task_ctx = await self.context.get_task_context(version, task_id)
        task = task_ctx['task']

        self.io.print_info(f"[{task_id}] {task.get('title', '')}")

        if self._execution_loop is None:
            # ExecutionLoop pas encore injecté (avant Phase 3) — stub
            self.io.print_warning('ExecutionLoop non disponible — marque la tâche manuellement.')
            done = self.io.prompt('Marquer comme terminée ? [o/N]', default='n').lower()
            if done in ('o', 'oui', 'y', 'yes'):
                await self.tasks.mark_done(version, task_id)
                self.context.invalidate_cache()
            return {'completed': False}

        # Déléguer à ExecutionLoop
        result = await self._execution_loop.run(task_ctx)

        if result.get('success'):
            await self.tasks.mark_done(version, task_id)
            self.context.invalidate_cache()
            self.io.print_success(f'{task_id} terminée.')
        else:
            await self.tasks.mark_failed(version, task_id, result.get('error', 'unknown'))
            self.io.print_error(f'{task_id} échouée : {result.get("error", "")}')

        # Vérifier si toutes les tâches sont done
        updated_ctx = await self.context.get_version_context(version)
        if not updated_ctx.get('pending_tasks') and not updated_ctx.get('failed_tasks'):
            return {'completed': True}

        return {'completed': False}

    async def _next_ready_task(
        self, version: str, pending: list[str], done: list[str]
    ) -> str | None:
        """Retourner le premier task_id dont toutes les dépendances sont satisfaites"""
        done_set = set(done)
        for task_id in pending:
            task = await self.tasks.get_task(version, task_id)
            if not task:
                continue
            deps = set(task.get('dependencies') or [])
            if deps.issubset(done_set):
                return task_id
        return None

    async def get_status(self) -> dict:
        """Retourner l'état courant de la phase d'exécution"""
        project = await self.memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            return {'version': None, 'pending': [], 'done': [], 'failed': []}

        ctx = await self.context.get_version_context(version)
        return {
            'version': version,
            'pending': ctx.get('pending_tasks', []),
            'done': ctx.get('done_tasks', []),
            'failed': ctx.get('failed_tasks', []),
            'deferred': ctx.get('deferred_tasks', []),
        }
```

## Séquence d'Exécution

```
ExecutionPhase.run()
  │
  ├─ get_project() → active_version
  ├─ get_version_context(version) → pending_tasks
  ├─ _next_ready_task() → task_id (dépendances satisfaites)
  ├─ get_task_context(version, task_id) → task + fichiers + décisions + skills
  │
  └─ ExecutionLoop.run(task_ctx)   ← Phase 3
       ├─ PromptBuilder.generate_code()
       ├─ LLMProvider.ask(role='code_generation')
       ├─ Appliquer les fichiers générés
       ├─ build_validate → si échec → retry (max 3)
       │    └─ PromptBuilder.generate_code_retry()
       └─ mark_done OU mark_failed
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `run()` lit `active_version` depuis `project.json` | ⬜ |
| 2 | `run()` retourne `completed: True` si aucune tâche `pending` | ⬜ |
| 3 | `_next_ready_task()` respecte l'ordre des dépendances | ⬜ |
| 4 | `get_task_context()` est appelé avec `version` et `task_id` (niveau 3) | ⬜ |
| 5 | Sans `ExecutionLoop` injecté, la phase fonctionne en mode stub | ⬜ |
| 6 | `mark_done()` est appelé sur la tâche après succès | ⬜ |
| 7 | `invalidate_cache()` est appelé après `mark_done()` ou `mark_failed()` | ⬜ |
| 8 | `get_status()` retourne `pending`, `done`, `failed`, `deferred` | ⬜ |
