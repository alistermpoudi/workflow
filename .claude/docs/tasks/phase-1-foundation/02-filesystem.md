# Phase 1 — Tâche 1.2 : FileSystem.py

## Objectif

Créer le module `FileSystem.py` qui encapsule toutes les opérations sur les fichiers `.workflow/`. C'est la couche basse sur laquelle tous les autres modules s'appuient. Toutes les opérations sont async et gèrent les erreurs proprement.

## Fichiers à Créer / Modifier

- `src/workflow/tools/filesystem.py` [CRÉER]
- `tests/unit/test_filesystem.py` [CRÉER]

## Responsabilités

- Initialiser la structure `.workflow/` dans un projet cible
- Lire/écrire des fichiers JSON (project.json, progress.json, etc.)
- Lire/écrire des fichiers Markdown (vision.md, TASK-XXX.md)
- Vérifier l'existence de fichiers et dossiers
- Lister les fichiers de tâches d'une version
- Opérations atomiques (écrire dans un fichier temporaire puis renommer)

## Implémentation

```python
# src/workflow/tools/filesystem.py
import json
from pathlib import Path
from dataclasses import dataclass, field
import aiofiles
import aiofiles.os


@dataclass
class WorkflowPaths:
    """Tous les chemins standards du dossier .workflow/"""
    workflow_dir: Path

    @property
    def project(self) -> Path:
        return self.workflow_dir / 'project.json'

    @property
    def vision(self) -> Path:
        return self.workflow_dir / 'vision.md'

    @property
    def features(self) -> Path:
        return self.workflow_dir / 'features.json'

    @property
    def tech_stack(self) -> Path:
        return self.workflow_dir / 'tech-stack.json'

    @property
    def code_index(self) -> Path:
        return self.workflow_dir / 'code-index.json'

    @property
    def decisions_db(self) -> Path:
        return self.workflow_dir / 'decisions.db'

    @property
    def failure_patterns(self) -> Path:
        return self.workflow_dir / 'failure-patterns.json'

    @property
    def design(self) -> Path:
        return self.workflow_dir / 'design.json'

    @property
    def questions_dir(self) -> Path:
        return self.workflow_dir / 'questions'

    @property
    def briefings_dir(self) -> Path:
        return self.workflow_dir / 'briefings'

    @property
    def skills_dir(self) -> Path:
        return self.workflow_dir / 'skills'

    def question_file(self, name: str) -> Path:
        return self.workflow_dir / 'questions' / name

    def briefing_file(self, date_str: str) -> Path:
        return self.workflow_dir / 'briefings' / f'{date_str}.md'

    def version_dir(self, version: str) -> Path:
        return self.workflow_dir / 'versions' / version

    def version_meta(self, version: str) -> Path:
        return self.workflow_dir / 'versions' / version / 'meta.json'

    def version_progress(self, version: str) -> Path:
        return self.workflow_dir / 'versions' / version / 'progress.json'

    def task_file(self, version: str, task_id: str) -> Path:
        return self.workflow_dir / 'versions' / version / 'tasks' / f'{task_id}.md'

    def tasks_dir(self, version: str) -> Path:
        return self.workflow_dir / 'versions' / version / 'tasks'


class FileSystem:
    def __init__(self, project_root: str):
        self.project_root = Path(project_root)
        self.workflow_dir = self.project_root / '.workflow'
        self.paths = WorkflowPaths(self.workflow_dir)

    async def init(self):
        """Initialiser la structure .workflow/ complète"""
        for directory in [
            self.workflow_dir,
            self.workflow_dir / 'versions',
            self.paths.questions_dir,
            self.paths.briefings_dir,
            self.paths.skills_dir,
        ]:
            directory.mkdir(parents=True, exist_ok=True)

    def exists(self, file_path: str | Path) -> bool:
        """Vérifier si un fichier ou dossier existe (synchrone — utilisé par SyncChecker)"""
        return Path(file_path).exists()

    def is_initialized(self) -> bool:
        """Vérifier si .workflow/ existe"""
        return self.workflow_dir.exists()

    async def read_json(self, file_path: str | Path) -> dict | None:
        """Lire un fichier JSON — retourne None si absent"""
        try:
            async with aiofiles.open(file_path, 'r', encoding='utf-8') as f:
                content = await f.read()
            return json.loads(content)
        except FileNotFoundError:
            return None
        except json.JSONDecodeError as e:
            raise ValueError(f'JSON invalide dans {file_path}: {e}') from e

    async def write_json(self, file_path: str | Path, data: dict):
        """Écrire un fichier JSON de façon atomique (tmp → rename)"""
        path = Path(file_path)
        path.parent.mkdir(parents=True, exist_ok=True)
        tmp = path.with_suffix('.tmp')
        async with aiofiles.open(tmp, 'w', encoding='utf-8') as f:
            await f.write(json.dumps(data, indent=2, ensure_ascii=False))
        tmp.rename(path)  # Atomique sur le même filesystem

    async def read_markdown(self, file_path: str | Path) -> str | None:
        """Lire un fichier Markdown — retourne None si absent"""
        try:
            async with aiofiles.open(file_path, 'r', encoding='utf-8') as f:
                return await f.read()
        except FileNotFoundError:
            return None

    async def write_markdown(self, file_path: str | Path, content: str):
        """Écrire un fichier Markdown de façon atomique"""
        path = Path(file_path)
        path.parent.mkdir(parents=True, exist_ok=True)
        tmp = path.with_suffix(path.suffix + '.tmp')
        async with aiofiles.open(tmp, 'w', encoding='utf-8') as f:
            await f.write(content)
        tmp.rename(path)

    async def read_selective(self, file_paths: list[str]) -> dict[str, str | None]:
        """Lire sélectivement un ensemble de fichiers source en parallèle (pour ContextManager)"""
        import asyncio

        async def read_one(fp: str) -> tuple[str, str | None]:
            try:
                async with aiofiles.open(self.project_root / fp, 'r', encoding='utf-8') as f:
                    return fp, await f.read()
            except FileNotFoundError:
                return fp, None

        results = await asyncio.gather(*[read_one(fp) for fp in file_paths])
        return dict(results)

    async def list_task_ids(self, version: str) -> list[str]:
        """Lister les tâches d'une version (retourne les IDs triés : TASK-001, TASK-002...)"""
        tasks_dir = self.paths.tasks_dir(version)
        if not tasks_dir.exists():
            return []
        return sorted([
            f.stem for f in tasks_dir.iterdir()
            if f.suffix == '.md'
        ])

    async def list_versions(self) -> list[str]:
        """Lister les versions disponibles"""
        versions_dir = self.workflow_dir / 'versions'
        if not versions_dir.exists():
            return []
        return sorted([
            d.name for d in versions_dir.iterdir()
            if d.is_dir()
        ])
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `init()` crée `.workflow/versions/`, `questions/`, `briefings/` et `skills/` | ⬜ |
| 2 | `read_json` retourne `None` pour un fichier absent (pas d'exception) | ⬜ |
| 3 | `write_json` est atomique (passe par un fichier `.tmp`) | ⬜ |
| 4 | `read_selective` lit plusieurs fichiers en parallèle | ⬜ |
| 5 | `list_task_ids` retourne les IDs triés | ⬜ |
| 6 | `exists()` retourne `True` pour un fichier présent, `False` pour un absent | ⬜ |
| 7 | `paths.decisions_db` pointe vers `.workflow/decisions.db` | ⬜ |
| 8 | `paths.failure_patterns` pointe vers `.workflow/failure-patterns.json` | ⬜ |
| 9 | `paths.skills_dir` pointe vers `.workflow/skills/` | ⬜ |
| 10 | Tests unitaires couvrent les cas normaux et les fichiers absents | ⬜ |
