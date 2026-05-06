# Phase 4 — Tâche 4.2 : VersionManager.py + GitManager.py (complet)

## Objectif

Créer `VersionManager.py` qui gère le cycle de vie des versions (création, switch, complétion) et `GitManager.py` qui encapsule toutes les opérations Git via `asyncio.create_subprocess_exec`. **Règle absolue : jamais de stash automatique.**

## Dépendances

- Phase 1 complète ✅

## Fichiers à Créer

- `src/workflow/tools/version_manager.py` [CRÉER]
- `src/workflow/tools/git_manager.py` [CRÉER]

---

## GitManager.py

```python
# src/workflow/tools/git_manager.py
import asyncio
from pathlib import Path


class GitManager:
    def __init__(self, project_root: str):
        self.root = Path(project_root).resolve()

    async def _run(self, *args: str) -> dict:
        """Exécuter une commande git et retourner exit_code + output"""
        proc = await asyncio.create_subprocess_exec(
            'git', *args,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.STDOUT,
            cwd=str(self.root),
        )
        stdout, _ = await proc.communicate()
        return {
            'exit_code': proc.returncode,
            'output': stdout.decode('utf-8', errors='replace').strip(),
        }

    async def is_clean(self) -> bool:
        """Vérifier si le working directory est propre (aucune modif non commitée)"""
        result = await self._run('status', '--porcelain')
        return result['exit_code'] == 0 and result['output'] == ''

    async def current_branch(self) -> str:
        result = await self._run('rev-parse', '--abbrev-ref', 'HEAD')
        return result['output'] if result['exit_code'] == 0 else ''

    async def branch_exists(self, branch: str) -> bool:
        result = await self._run('branch', '--list', branch)
        return branch in result['output']

    async def create_branch(self, branch: str) -> dict:
        return await self._run('checkout', '-b', branch)

    async def switch_branch(self, branch: str) -> dict:
        return await self._run('checkout', branch)

    async def add_all(self) -> dict:
        return await self._run('add', '-A')

    async def commit(self, message: str) -> dict:
        return await self._run('commit', '-m', message)

    async def push(self, branch: str | None = None) -> dict:
        if branch:
            return await self._run('push', '-u', 'origin', branch)
        return await self._run('push')

    async def log(self, n: int = 10) -> str:
        result = await self._run('log', f'--oneline', f'-{n}')
        return result['output']

    async def diff_stat(self) -> str:
        result = await self._run('diff', '--stat', 'HEAD')
        return result['output']
```

---

## VersionManager.py

```python
# src/workflow/tools/version_manager.py
from pathlib import Path

from workflow.core.project_memory import ProjectMemory
from workflow.tools.git_manager import GitManager


BRANCH_PREFIX = 'workflow/'


class VersionManager:
    def __init__(self, project_root: str):
        self.memory = ProjectMemory(project_root)
        self.git = GitManager(project_root)

    async def list_versions(self) -> list[dict]:
        """Lister toutes les versions avec leur statut"""
        versions_dir = Path(self.memory.paths.versions_dir)
        if not versions_dir.exists():
            return []

        result = []
        for version_dir in sorted(versions_dir.iterdir()):
            if not version_dir.is_dir():
                continue
            meta_file = version_dir / 'meta.json'
            if meta_file.exists():
                import json
                meta = json.loads(meta_file.read_text())
                result.append(meta)
        return result

    async def create(self, name: str, description: str = '') -> dict:
        """Créer une nouvelle version avec sa branche git"""
        project = await self.memory.get_project()
        active = (project or {}).get('active_version')

        # Une seule version ACTIVE à la fois
        if active:
            active_meta = await self.memory.get_version_meta(active)
            if (active_meta or {}).get('status') not in ('COMPLETED', 'ARCHIVED'):
                return {
                    'ok': False,
                    'error': f'La version {active} est toujours active. Complète-la avant d\'en créer une nouvelle.',
                }

        # Créer la branche git
        branch = f'{BRANCH_PREFIX}{name}'
        if not await self.git.branch_exists(branch):
            result = await self.git.create_branch(branch)
            if result['exit_code'] != 0:
                return {'ok': False, 'error': f'Impossible de créer la branche : {result["output"]}'}

        # Créer les fichiers de version
        await self.memory.init_version(name, {
            'name': name,
            'description': description,
            'status': 'ACTIVE',
            'branch': branch,
        })

        await self.memory.update_project({'active_version': name})
        return {'ok': True, 'version': name, 'branch': branch}

    async def switch(self, version: str) -> dict:
        """Changer la version active — bloque si repo non propre"""
        if not await self.git.is_clean():
            return {
                'ok': False,
                'blocked': True,
                'reason': (
                    'Le répertoire de travail n\'est pas propre. '
                    'Commite tes modifications avant de changer de version. '
                    'Workflow ne stashe jamais automatiquement.'
                ),
            }

        meta = await self.memory.get_version_meta(version)
        if not meta:
            return {'ok': False, 'error': f'Version {version} introuvable.'}

        branch = meta.get('branch', f'{BRANCH_PREFIX}{version}')
        result = await self.git.switch_branch(branch)
        if result['exit_code'] != 0:
            return {'ok': False, 'error': f'git checkout échoué : {result["output"]}'}

        await self.memory.update_project({'active_version': version})
        return {'ok': True, 'version': version, 'branch': branch}

    async def complete(self) -> dict:
        """Marquer la version active comme COMPLETED"""
        project = await self.memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            return {'ok': False, 'error': 'Aucune version active.'}

        progress = await self.memory.get_progress(version)
        pending = (progress or {}).get('pending', [])
        if pending:
            return {
                'ok': False,
                'error': f'{len(pending)} tâche(s) encore en attente : {", ".join(pending)}',
            }

        await self.memory.update_version_status(version, 'COMPLETED')
        return {'ok': True, 'version': version, 'status': 'COMPLETED'}

    async def create_hotfix(self, name: str, reason: str) -> dict:
        """Créer un hotfix — bloque si repo non propre"""
        if not await self.git.is_clean():
            return {
                'ok': False,
                'blocked': True,
                'reason': 'Commite tes modifications avant de créer un hotfix.',
            }

        hotfix_branch = f'{BRANCH_PREFIX}hotfix/{name}'
        result = await self.git.create_branch(hotfix_branch)
        if result['exit_code'] != 0:
            return {'ok': False, 'error': result['output']}

        return {'ok': True, 'branch': hotfix_branch, 'reason': reason}
```

---

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `GitManager._run()` utilise `asyncio.create_subprocess_exec` (pas shell=True) | ⬜ |
| 2 | `GitManager.is_clean()` retourne `True` seulement si `git status --porcelain` est vide | ⬜ |
| 3 | `VersionManager.switch()` retourne `{'blocked': True, 'reason': ...}` si repo non propre | ⬜ |
| 4 | `VersionManager.switch()` ne stashe jamais automatiquement | ⬜ |
| 5 | `VersionManager.create()` refuse de créer si une version ACTIVE existe déjà | ⬜ |
| 6 | `VersionManager.create()` crée la branche `workflow/vX.Y` avant le meta.json | ⬜ |
| 7 | `VersionManager.complete()` refuse si des tâches sont encore `pending` | ⬜ |
| 8 | `create_hotfix()` bloque si repo non propre (même règle que switch) | ⬜ |
| 9 | La branche du hotfix suit le pattern `workflow/hotfix/<name>` | ⬜ |
