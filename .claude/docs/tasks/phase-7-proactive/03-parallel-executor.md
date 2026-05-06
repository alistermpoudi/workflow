# Phase 3 — Tâche 3.6 : ParallelExecutor.py

## Objectif

Exécuter plusieurs tâches **indépendantes** en parallèle via des `git worktrees` isolés et `asyncio.gather()`. Quand le graphe de dépendances révèle 3 tâches sans lien entre elles, Workflow lance 3 instances `ExecutionLoop` simultanément et réduit le temps de livraison d'un facteur N.

> **Aucun outil de codage ne fait ça aujourd'hui.** C'est le différenciateur le plus fort de Workflow sur les projets réels.

## Dépendances

- Tâches 3.1 ✅ (`ExecutionLoop`)
- `GitManager` (Phase 1) ✅

## Fichiers à Créer

- `src/workflow/tools/parallel_executor.py` [CRÉER]

## Concept : Git Worktrees

Chaque tâche parallèle tourne dans un worktree git isolé — une copie du repo sur sa propre branche, sans interférer avec les autres :

```
.workflow/worktrees/
  TASK-005/    ← git worktree sur branche workflow/v1.0/TASK-005
  TASK-006/    ← git worktree sur branche workflow/v1.0/TASK-006
  TASK-008/    ← git worktree sur branche workflow/v1.0/TASK-008
```

Une fois toutes les tâches terminées, Workflow merge les branches dans la branche principale de la version.

## Algorithme de Planification

```
1. Charger toutes les tâches pending de la version
2. Construire le graphe de dépendances
3. Identifier les tâches "ready" (dépendances toutes DONE)
4. Si ready >= 2 → lancer en parallèle (max MAX_PARALLEL workers)
5. asyncio.gather() sur les ExecutionLoop
6. Pour chaque tâche réussie → merge dans la branche version
7. Relancer le calcul des tâches ready (certaines le deviennent après merge)
8. Répéter jusqu'à épuisement des tâches
```

## Implémentation

```python
# src/workflow/tools/parallel_executor.py
import asyncio
from pathlib import Path
from datetime import datetime, timezone

from workflow.core.project_memory import ProjectMemory
from workflow.core.context_manager import ContextManager
from workflow.tools.task_manager import TaskManager
from workflow.tools.git_manager import GitManager
from workflow.tools.execution_loop import ExecutionLoop
from workflow.llm.llm_provider import LLMProvider

MAX_PARALLEL = 3
WORKTREES_DIR = '.workflow/worktrees'


class ParallelExecutor:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.project_root = Path(project_root).resolve()
        self.memory = ProjectMemory(project_root)
        self.tasks = TaskManager(project_root)
        self.git = GitManager(project_root)
        self.llm = llm
        self.io = io

    async def run_parallel(self) -> dict:
        """
        Exécuter toutes les tâches pending en parallélisant les tâches indépendantes.
        Retourne {'completed': True} quand toutes les tâches de la version sont done.
        """
        project = await self.memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            return {'completed': False, 'error': 'Aucune version active.'}

        tech_stack = await self.memory.get_tech_stack() or {}
        total_done = 0
        total_failed = 0

        while True:
            progress = await self.memory.get_progress(version)
            pending = progress.get('pending', [])
            done = set(progress.get('done', []))

            if not pending:
                self.io.print_success(
                    f'Toutes les tâches terminées — {len(done)} done, {total_failed} failed.'
                )
                return {'completed': True, 'done': total_done, 'failed': total_failed}

            # Identifier les tâches "ready" (dépendances satisfaites)
            ready = await self._get_ready_tasks(version, pending, done)
            if not ready:
                self.io.print_warning('Aucune tâche prête — dépendances circulaires ou failed.')
                return {'completed': False, 'pending': pending}

            # Limiter le parallélisme
            batch = ready[:MAX_PARALLEL]

            if len(batch) == 1:
                self.io.print_info(f'Exécution séquentielle : {batch[0]}')
            else:
                self.io.print_info(
                    f'Exécution parallèle : {", ".join(batch)} ({len(batch)} workers)'
                )

            # Préparer les worktrees
            worktree_paths = await self._setup_worktrees(version, batch)

            # Lancer en parallèle
            results = await asyncio.gather(
                *[
                    self._run_in_worktree(version, task_id, wt_path, tech_stack)
                    for task_id, wt_path in zip(batch, worktree_paths)
                ],
                return_exceptions=True,
            )

            # Traiter les résultats
            for task_id, result in zip(batch, results):
                if isinstance(result, Exception):
                    await self.tasks.mark_failed(version, task_id, str(result))
                    total_failed += 1
                    self.io.print_error(f'{task_id} — exception : {result}')
                elif result.get('success'):
                    await self._merge_worktree(version, task_id)
                    await self.tasks.mark_done(version, task_id)
                    total_done += 1
                    self.io.print_success(f'{task_id} ✓')
                else:
                    await self.tasks.mark_failed(version, task_id, result.get('error', '?'))
                    total_failed += 1
                    self.io.print_error(f'{task_id} ✗ — {result.get("error", "")[:100]}')

            # Nettoyer les worktrees
            await self._cleanup_worktrees(batch)

    async def _get_ready_tasks(
        self, version: str, pending: list[str], done: set[str]
    ) -> list[str]:
        """Retourner les tâches dont toutes les dépendances sont satisfaites"""
        ready = []
        for task_id in pending:
            task = await self.tasks.get_task(version, task_id)
            if not task:
                continue
            deps = set(task.get('dependencies') or [])
            if deps.issubset(done):
                ready.append(task_id)
        return ready

    async def _setup_worktrees(self, version: str, task_ids: list[str]) -> list[Path]:
        """Créer un git worktree isolé pour chaque tâche"""
        paths = []
        worktrees_base = self.project_root / WORKTREES_DIR
        worktrees_base.mkdir(parents=True, exist_ok=True)

        for task_id in task_ids:
            branch = f'workflow/{version}/{task_id.lower()}'
            wt_path = worktrees_base / task_id

            # Supprimer le worktree précédent si existant
            if wt_path.exists():
                await self.git._run('worktree', 'remove', '--force', str(wt_path))

            # Créer la branche et le worktree
            await self.git._run('branch', branch)
            await self.git._run('worktree', 'add', str(wt_path), branch)
            paths.append(wt_path)

        return paths

    async def _run_in_worktree(
        self, version: str, task_id: str, wt_path: Path, tech_stack: dict
    ) -> dict:
        """Exécuter une tâche dans son worktree isolé"""
        wt_str = str(wt_path)

        # Créer un ExecutionLoop pointant sur le worktree
        wt_memory = ProjectMemory(wt_str)
        ctx = ContextManager(wt_str, self.llm)
        loop = ExecutionLoop(wt_str, self.llm, self.io, tech_stack)

        try:
            task_ctx = await ctx.get_task_context(version, task_id)
        except ValueError:
            # Les fichiers .workflow/ sont dans le projet principal, pas dans le worktree
            # Charger depuis le projet principal
            main_ctx = ContextManager(str(self.project_root), self.llm)
            task_ctx = await main_ctx.get_task_context(version, task_id)

        return await loop.run(task_ctx)

    async def _merge_worktree(self, version: str, task_id: str) -> None:
        """Merger le worktree d'une tâche réussie dans la branche version"""
        branch = f'workflow/{version}/{task_id.lower()}'
        version_branch = f'workflow/{version}'

        # Commiter dans le worktree
        wt_path = self.project_root / WORKTREES_DIR / task_id
        wt_git = GitManager(str(wt_path))
        await wt_git._run('add', '-A')
        await wt_git._run('commit', '-m', f'feat({task_id}): implémentation automatique')

        # Merger dans la branche version
        await self.git._run('merge', '--no-ff', branch, '-m', f'Merge {task_id}')

    async def _cleanup_worktrees(self, task_ids: list[str]) -> None:
        """Supprimer les worktrees après merge"""
        for task_id in task_ids:
            wt_path = self.project_root / WORKTREES_DIR / task_id
            await self.git._run('worktree', 'remove', '--force', str(wt_path))
            branch = f'workflow/TASK-{task_id.split("-")[-1].lower()}'
            await self.git._run('branch', '-d', branch)
```

## Visualisation du Parallélisme

```
Version v1.0 — tâches pending : TASK-001, TASK-002, TASK-003, TASK-004, TASK-005

Graphe de dépendances :
  TASK-001 (setup)
  TASK-002 (tests) → dépend de TASK-001
  TASK-003 (auth) → dépend de TASK-001, TASK-002
  TASK-004 (dashboard UI) → dépend de TASK-001
  TASK-005 (API endpoints) → dépend de TASK-001

Cycle 1 : TASK-001 seule (dépendance de tout)
Cycle 2 : TASK-002, TASK-004, TASK-005 en parallèle ← 3 workers simultanés
Cycle 3 : TASK-003 seule (dépend de TASK-002 terminé)

Temps séquentiel : 5 tâches × ~30min = 2h30
Temps parallèle  : 3 cycles × ~30min = 1h30  ← gain réel : -40%
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `_get_ready_tasks()` retourne uniquement les tâches dont les dépendances sont dans `done` | ⬜ |
| 2 | `_setup_worktrees()` crée un dossier isolé par tâche dans `.workflow/worktrees/` | ⬜ |
| 3 | `asyncio.gather()` lance les ExeuctionLoop en parallèle réel | ⬜ |
| 4 | Une exception dans un worker n'annule pas les autres workers | ⬜ |
| 5 | `_merge_worktree()` commit dans le worktree avant de merger | ⬜ |
| 6 | `_cleanup_worktrees()` supprime les worktrees après merge | ⬜ |
| 7 | `MAX_PARALLEL = 3` est respecté même si 10 tâches sont prêtes | ⬜ |
| 8 | La boucle se termine quand `pending` est vide | ⬜ |
| 9 | La boucle s'arrête avec un message si aucune tâche ready (dépendances circulaires) | ⬜ |
