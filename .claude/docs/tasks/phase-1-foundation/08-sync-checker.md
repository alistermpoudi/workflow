# Phase 1 — Tâche 1.6 : SyncChecker.py

## Objectif

Créer `SyncChecker.py` et `GitManager.py` — la pièce qui garantit la promesse principale de Workflow : reprendre exactement où on s'était arrêté après un context overflow ou une interruption. Vérifie la cohérence entre l'état Git et l'état `.workflow/`, et détecte les modifications manuelles.

## Dépendances

- Tâche 1.2 ✅ (`FileSystem.py`)
- Tâche 1.3 ✅ (`ProjectMemory.py`)

## Fichiers à Créer / Modifier

- `src/workflow/tools/git_manager.py` [CRÉER]
- `src/workflow/core/sync_checker.py` [CRÉER]
- `tests/unit/test_sync_checker.py` [CRÉER]

## Implémentation — `GitManager.py`

```python
# src/workflow/tools/git_manager.py
import asyncio
from pathlib import Path


class GitManager:
    def __init__(self, project_root: str):
        self.project_root = project_root

    async def _run(self, *args: str) -> str:
        """Exécution sûre via create_subprocess_exec (pas d'injection shell possible)"""
        proc = await asyncio.create_subprocess_exec(
            *args,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            cwd=self.project_root,
        )
        stdout, stderr = await proc.communicate()
        if proc.returncode != 0:
            raise RuntimeError(stderr.decode().strip() or stdout.decode().strip())
        return stdout.decode().strip()

    async def current_branch(self) -> str:
        return await self._run('git', 'rev-parse', '--abbrev-ref', 'HEAD')

    async def is_clean(self) -> bool:
        result = await self._run('git', 'status', '--porcelain')
        return result == ''

    async def get_modified_since(self, since: str) -> list[str]:
        """Fichiers modifiés depuis un timestamp ISO"""
        result = await self._run(
            'git', 'log', f'--since={since}',
            '--name-only', '--format=', '--diff-filter=M',
        )
        return [f for f in result.split('\n') if f.strip()] if result else []

    async def get_diff(self, files: list[str] | None = None) -> str:
        if files:
            return await self._run('git', 'diff', 'HEAD', '--', *files)
        return await self._run('git', 'diff', 'HEAD')

    async def branch_exists(self, branch: str) -> bool:
        """Sûr — branch passé via exec, pas interpolé dans un shell"""
        try:
            await self._run('git', 'rev-parse', '--verify', branch)
            return True
        except RuntimeError:
            return False

    async def get_uncommitted_files(self) -> list[str]:
        """Fichiers modifiés non-commités (staged + unstaged, hors .workflow/)"""
        result = await self._run('git', 'status', '--porcelain')
        if not result:
            return []
        return [
            line[3:].strip()
            for line in result.split('\n')
            if line and not line[3:].strip().startswith('.workflow/')
        ]

    async def is_git_repo(self) -> bool:
        try:
            await self._run('git', 'rev-parse', '--git-dir')
            return True
        except (RuntimeError, FileNotFoundError):
            return False

    async def checkout(self, branch: str, create: bool = False):
        args = ['git', 'checkout']
        if create:
            args.append('-b')
        args.append(branch)
        await self._run(*args)

    async def commit(self, message: str, files: list[str] | None = None):
        """Commit sûr — message passé via exec, jamais interpolé dans un shell"""
        if files:
            await self._run('git', 'add', *files)
        else:
            await self._run('git', 'add', '-A')
        await self._run('git', 'commit', '-m', message)

    async def merge(self, branch: str):
        await self._run('git', 'merge', '--no-ff', branch)
```

## Implémentation — `SyncChecker.py`

```python
# src/workflow/core/sync_checker.py
from pathlib import Path
from workflow.tools.git_manager import GitManager
from workflow.core.project_memory import ProjectMemory
from workflow.tools.filesystem import FileSystem


class SyncChecker:
    def __init__(self, project_root: str):
        self.project_root = project_root
        self.git = GitManager(project_root)
        self.memory = ProjectMemory(project_root)
        self.fs = FileSystem(project_root)

    async def check(self) -> dict:
        """Vérification complète au démarrage d'une session"""
        if not await self.git.is_git_repo():
            return {'type': 'NO_GIT', 'message': 'Pas de repo Git détecté dans ce projet.'}

        project = await self.memory.get_project()
        if not project:
            return {'type': 'NOT_INITIALIZED', 'message': 'Aucun projet Workflow trouvé dans .workflow/'}

        # 1. Vérifier cohérence branche / version active
        active_version = project.get('current_version')
        if active_version:
            current_branch = await self.git.current_branch()
            version_meta = await self.memory.get_version_meta(active_version)
            # Lire meta.branch — les hotfixes ont 'workflow/hotfix/vX.Y.Z'
            expected_branch = (version_meta or {}).get('branch', f'workflow/{active_version}')

            if (
                current_branch != expected_branch
                and current_branch not in ('main', 'master')
            ):
                return {
                    'type': 'BRANCH_MISMATCH',
                    'message': (
                        f'Branche Git "{current_branch}" ≠ version active "{active_version}" '
                        f'(attendu: {expected_branch})'
                    ),
                    'current_branch': current_branch,
                    'expected_branch': expected_branch,
                    'active_version': active_version,
                }

        # 2. Détecter modifications depuis la dernière session
        last_session = await self.memory.get_last_session_timestamp()
        committed_changes = (
            await self.git.get_modified_since(last_session.isoformat())
            if last_session else []
        )
        uncommitted_changes = await self.git.get_uncommitted_files()
        modified_files = list(set(committed_changes + uncommitted_changes))

        if modified_files:
            diff = await self.git.get_diff(modified_files)
            return {
                'type': 'MANUAL_CHANGES',
                'message': f'{len(modified_files)} fichier(s) modifié(s) depuis la dernière session.',
                'files': modified_files,
                'diff': diff,
                'last_session': last_session.isoformat() if last_session else None,
                'active_version': project.get('current_version'),
            }

        # 3. Vérifier que le répertoire est propre
        if not await self.git.is_clean():
            return {
                'type': 'DIRTY_REPO',
                'message': 'Des modifications non commitées existent dans le répertoire.',
                'diff': await self.git.get_diff(),
            }

        return {
            'type': 'CLEAN',
            'message': (
                f"Projet \"{project['name']}\" — "
                + (f"v{active_version} ACTIVE" if active_version else "pas de version active")
                + "."
            ),
            'project': project,
        }

    async def check_before_switch(self) -> dict:
        """Vérifier avant un switch de version (bloquer si repo non propre)"""
        if not await self.git.is_clean():
            diff = await self.git.get_diff()
            return {
                'can_switch': False,
                'message': 'Répertoire non propre. Commite tes changements avant de changer de version.',
                'diff': diff,
            }
        return {'can_switch': True}

    async def check_preconditions(self, task: dict, version: str) -> dict:
        """Vérifier les préconditions déclaratives d'une tâche avant de la démarrer"""
        preconditions = task.get('preconditions')
        if not preconditions:
            return {'met': True}

        failures: list[str] = []

        # Vérifier les fichiers requis
        for fp in (preconditions.get('filesExist') or []):
            if not self.fs.exists(Path(self.project_root) / fp):
                failures.append(f'Fichier requis absent : {fp}')

        # Vérifier la branche Git
        required_branch = preconditions.get('branch')
        if required_branch:
            current_branch = await self.git.current_branch()
            if current_branch != required_branch:
                failures.append(
                    f'Branche requise : "{required_branch}", actuelle : "{current_branch}"'
                )

        # Vérifier les tâches dépendantes
        for dep_id in (preconditions.get('tasksCompleted') or []):
            progress = await self.memory.get_progress(version)
            if dep_id not in progress['done']:
                failures.append(f'Tâche dépendante non terminée : {dep_id}')

        return {'met': len(failures) == 0} if not failures else {'met': False, 'failures': failures}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `check()` retourne `NOT_INITIALIZED` si pas de `.workflow/` | ⬜ |
| 2 | `check()` retourne `BRANCH_MISMATCH` si branche incorrecte | ⬜ |
| 3 | `check()` retourne `MANUAL_CHANGES` avec la liste des fichiers | ⬜ |
| 3b | `check()` retourne `MANUAL_CHANGES` même si les fichiers sont non-commités | ⬜ |
| 4 | `check()` retourne `CLEAN` si tout est cohérent | ⬜ |
| 5 | `check_before_switch()` bloque si repo non propre | ⬜ |
| 6 | `GitManager.is_git_repo()` retourne False hors d'un repo | ⬜ |
| 7 | `check_preconditions()` retourne les échecs précis | ⬜ |
| 8 | `check_preconditions()` retourne `{ met: True }` si toutes passent | ⬜ |
| 9 | Tests mockent les commandes git via `pytest-mock` | ⬜ |
| 10 | `GitManager._run()` utilise `create_subprocess_exec` — pas d'injection shell | ⬜ |
| 11 | Un message contenant `$(rm -rf /)` est passé littéralement à git, sans interprétation | ⬜ |
| 12 | `check()` utilise `meta.branch` pour comparer — pas de faux MISMATCH sur les hotfixes | ⬜ |
| 13 | `check()` retourne `active_version` dans le rapport `MANUAL_CHANGES` | ⬜ |
