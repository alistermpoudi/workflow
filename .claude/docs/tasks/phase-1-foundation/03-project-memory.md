# Phase 1 — Tâche 1.3 : ProjectMemory.py

## Objectif

Créer le module `ProjectMemory.py` — la couche de lecture/écriture de tous les fichiers de métadonnées du projet (sauf les tâches, gérées par `TaskManager`). C'est le module que tous les autres consultent pour connaître l'état du projet.

## Dépendances

- Tâche 1.2 ✅ (`FileSystem.py`)

## Fichiers à Créer / Modifier

- `src/workflow/core/project_memory.py` [CRÉER]
- `tests/unit/test_project_memory.py` [CRÉER]

## Responsabilités

- Créer un nouveau projet (initialiser tous les fichiers `.workflow/`)
- Lire/écrire `project.json`, `vision.md`, `features.json`, `tech-stack.json`
- Lire/écrire les `meta.json` et `progress.json` de chaque version
- Retourner un résumé court du projet (pour `ContextManager` — ~500 tokens max)
- Gérer la version active

## Implémentation

```python
# src/workflow/core/project_memory.py
from datetime import datetime, timezone
from workflow.tools.filesystem import FileSystem


class ProjectMemory:
    def __init__(self, project_root: str):
        self.fs = FileSystem(project_root)

    async def init_project(self, data: dict) -> dict:
        """Initialiser un nouveau projet"""
        await self.fs.init()
        now = datetime.now(timezone.utc).isoformat()
        project = {
            'name': data['name'],
            'description': data.get('description', ''),
            'created_at': now,
            'current_version': None,
            'status': 'DISCOVERY',  # DISCOVERY | SPECIFICATION | ARCHITECTURE | VALIDATION | ACTIVE
            'last_session_at': now,
        }
        await self.fs.write_json(self.fs.paths.project, project)
        return project

    async def get_project(self) -> dict | None:
        """Lire project.json"""
        return await self.fs.read_json(self.fs.paths.project)

    async def update_project(self, updates: dict) -> dict:
        """Mettre à jour des champs spécifiques de project.json"""
        current = await self.get_project()
        updated = {
            **(current or {}),
            **updates,
            'last_session_at': datetime.now(timezone.utc).isoformat(),
        }
        await self.fs.write_json(self.fs.paths.project, updated)
        return updated

    # vision.md
    async def get_vision(self) -> str | None:
        return await self.fs.read_markdown(self.fs.paths.vision)

    async def save_vision(self, content: str):
        await self.fs.write_markdown(self.fs.paths.vision, content)

    # features.json
    async def get_features(self) -> dict | None:
        return await self.fs.read_json(self.fs.paths.features)

    async def save_features(self, features: dict):
        await self.fs.write_json(self.fs.paths.features, features)

    # tech-stack.json
    async def get_tech_stack(self) -> dict | None:
        return await self.fs.read_json(self.fs.paths.tech_stack)

    async def save_tech_stack(self, stack: dict):
        await self.fs.write_json(self.fs.paths.tech_stack, stack)

    # Version meta
    async def get_version_meta(self, version: str) -> dict | None:
        return await self.fs.read_json(self.fs.paths.version_meta(version))

    async def save_version_meta(self, version: str, meta: dict):
        await self.fs.write_json(self.fs.paths.version_meta(version), meta)

    # Progress d'une version
    async def get_progress(self, version: str) -> dict:
        progress = await self.fs.read_json(self.fs.paths.version_progress(version))
        return progress or {'done': [], 'pending': [], 'failed': [], 'deferred': []}

    async def save_progress(self, version: str, progress: dict):
        await self.fs.write_json(self.fs.paths.version_progress(version), progress)

    # Version active
    async def get_active_version(self) -> str | None:
        project = await self.get_project()
        return project.get('current_version') if project else None

    # Timestamp de la dernière session (pour SyncChecker)
    async def get_last_session_timestamp(self) -> datetime | None:
        project = await self.get_project()
        if project and project.get('last_session_at'):
            return datetime.fromisoformat(project['last_session_at'])
        return None

    # Résumé court pour ContextManager (~500 tokens max)
    async def get_project_summary(self) -> dict | None:
        project = await self.get_project()
        stack = await self.get_tech_stack()
        if not project:
            return None
        return {
            'name': project['name'],
            'description': project.get('description', '')[:200],
            'status': project.get('status', 'DISCOVERY'),
            'current_version': project.get('current_version'),
            'language': stack.get('language', 'unknown') if stack else 'unknown',
            'framework': stack.get('framework', 'unknown') if stack else 'unknown',
        }

    # Lister toutes les versions
    async def list_versions(self) -> list[str]:
        return await self.fs.list_versions()

    # failure-patterns.json
    async def get_failure_patterns(self) -> list:
        return await self.fs.read_json(self.fs.paths.failure_patterns) or []

    async def save_failure_patterns(self, patterns: list):
        await self.fs.write_json(self.fs.paths.failure_patterns, patterns)

    # design.json — préférences visuelles collectées en DiscoveryPhase
    async def get_design(self) -> dict | None:
        return await self.fs.read_json(self.fs.paths.design)

    async def save_design(self, design: dict):
        await self.fs.write_json(self.fs.paths.design, design)
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `init_project()` crée tous les fichiers de base + dossiers `questions/`, `briefings/` et `skills/` | ⬜ |
| 2 | `get_project_summary()` retourne moins de 500 tokens (vérifier en test) | ⬜ |
| 3 | `update_project()` met à jour `last_session_at` automatiquement | ⬜ |
| 4 | `get_progress()` retourne la structure vide si fichier absent | ⬜ |
| 5 | `get_failure_patterns()` retourne une liste vide si le fichier n'existe pas encore | ⬜ |
| 6 | Tests unitaires couvrent init + lecture + écriture | ⬜ |
| 7 | `get_last_session_timestamp()` retourne `None` si `last_session_at` est absent | ⬜ |
| 8 | `init_project()` inclut `last_session_at` dans le schéma initial de `project.json` | ⬜ |
| 9 | `get_design()` retourne `None` si `design.json` n'existe pas encore | ⬜ |
| 10 | `save_design()` persiste correctement les préférences visuelles dans `design.json` | ⬜ |
