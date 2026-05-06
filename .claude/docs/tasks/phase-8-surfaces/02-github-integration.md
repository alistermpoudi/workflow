# Phase 5 — Tâche 5.2 : GitHubIntegration.py

## Objectif

Connecter Workflow aux événements GitHub/GitLab. Quand une PR est mergée, la tâche correspondante est automatiquement marquée DONE. Quand un CI casse, le daemon reçoit l'information et prépare une analyse.

## Dépendances

- Phase 4 (MCP Server) ✅
- Tâche 3.4 (DaemonHeartbeat) ✅

## Fichiers à Créer

- `src/workflow/interfaces/github_integration.py` [CRÉER]

## Événements Gérés

| Événement GitHub | Action Workflow |
|-----------------|-----------------|
| PR mergée | Cherche la tâche liée (par titre/branche) → marque DONE |
| CI failed | Notifie le daemon → analyse de l'erreur |
| Issue créée avec label `workflow` | Crée une tâche dans la version active |
| Release publiée | Marque la version comme COMPLETED |

## Liaison PR ↔ Tâche

La liaison se fait par convention de nommage :
- Branche : `workflow/v1.0/TASK-007` → lié à TASK-007
- Titre PR : `[TASK-007] API routes client` → lié à TASK-007
- Corps PR : `Closes TASK-007` → lié à TASK-007

## Implémentation

```python
# src/workflow/interfaces/github_integration.py
import re
from pathlib import Path
from datetime import datetime, timezone

from workflow.core.project_memory import ProjectMemory
from workflow.tools.task_manager import TaskManager


TASK_ID_PATTERN = re.compile(r'TASK-\d{3}')


class GitHubIntegration:
    def __init__(self, project_root: str):
        self.project_root = Path(project_root).resolve()
        self.memory = ProjectMemory(project_root)
        self.tasks = TaskManager(project_root)
        self._alerts_dir = self.project_root / '.workflow' / 'alerts'

    async def handle_pr_merged(self, pr: dict) -> dict:
        """Traiter une PR mergée — chercher et marquer la tâche associée"""
        task_id = self._extract_task_id(pr)
        if not task_id:
            return {'ok': True, 'task_id': None, 'message': 'Aucune tâche liée à cette PR.'}

        project = await self.memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            return {'ok': False, 'error': 'Aucune version active.'}

        task = await self.tasks.get_task(version, task_id)
        if not task:
            return {'ok': True, 'task_id': task_id, 'message': f'{task_id} introuvable — log silencieux.'}

        await self.tasks.mark_done(version, task_id)
        return {'ok': True, 'task_id': task_id, 'marked_done': True}

    async def handle_ci_failed(self, build: dict) -> dict:
        """Traiter un CI échoué — créer une alerte dans .workflow/alerts/"""
        self._alerts_dir.mkdir(parents=True, exist_ok=True)

        timestamp = datetime.now(timezone.utc).strftime('%Y%m%d-%H%M%S')
        alert_path = self._alerts_dir / f'ci-failed-{timestamp}.md'

        content = f"""# CI Failed — {timestamp}

**Branche** : {build.get('branch', '?')}
**Job** : {build.get('job', '?')}
**Erreur** :

```
{build.get('error', '(aucune sortie)')}
```
"""
        alert_path.write_text(content, encoding='utf-8')
        return {'ok': True, 'alert': str(alert_path)}

    async def handle_release_published(self, release: dict) -> dict:
        """Marquer la version active comme COMPLETED quand une release est publiée"""
        project = await self.memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            return {'ok': False, 'error': 'Aucune version active.'}

        await self.memory.update_version_status(version, 'COMPLETED')
        return {'ok': True, 'version': version, 'status': 'COMPLETED'}

    def _extract_task_id(self, pr: dict) -> str | None:
        """Extraire l'ID de tâche depuis la branche, le titre, ou le corps de la PR"""
        for field in ('head_branch', 'title', 'body'):
            value = pr.get(field) or ''
            match = TASK_ID_PATTERN.search(value)
            if match:
                return match.group(0)
        return None
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | PR mergée avec branche `workflow/*/TASK-XXX` → tâche marquée DONE | ⬜ |
| 2 | CI failed → fichier créé dans `.workflow/alerts/` avec l'erreur | ⬜ |
| 3 | La liaison fonctionne sur branche, titre ET corps de la PR | ⬜ |
| 4 | Si aucune tâche trouvée pour une PR → retourne `ok: True` sans erreur | ⬜ |
| 5 | `handle_release_published()` marque la version COMPLETED | ⬜ |
