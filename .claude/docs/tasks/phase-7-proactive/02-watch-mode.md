# Phase 3 — Tâche 3.5 : WatchMode.py

## Objectif

Créer `WatchMode.py` — surveillance passive des fichiers du projet via **`watchfiles.awatch`**. Quand un fichier est créé ou modifié **hors du scope de la tâche courante**, Workflow crée un fichier question dans `.workflow/questions/`. L'utilisateur répond en éditant le fichier. **Zéro interruption.**

## Dépendances

- Phase 1 complète ✅
- `watchfiles` dans les dépendances (`uv add watchfiles`)

## Fichiers à Créer

- `src/workflow/core/watch_mode.py` [CRÉER]

## Format des Fichiers Questions

```
.workflow/questions/
  QUESTION-001.md
  QUESTION-002.md
  ...
```

```markdown
# QUESTION-001

**Fichier modifié** : `src/utils/helpers.py`
**Hors scope de** : TASK-005 (qui ne touche que src/api/routes.py)

Cette modification semble hors scope de la tâche en cours.
Que veux-tu faire ?

- [ ] Créer une nouvelle tâche pour ce changement
- [ ] Inclure dans TASK-005
- [ ] Ignorer (modification manuelle intentionnelle)

**Réponse** : (édite cette ligne)
---
status: pending
created_at: 2026-04-05T10:23:00Z
```

## Implémentation

```python
# src/workflow/core/watch_mode.py
import asyncio
import re
from datetime import datetime, timezone
from pathlib import Path

from watchfiles import awatch, Change

from workflow.core.project_memory import ProjectMemory
from workflow.tools.task_manager import TaskManager


QUESTIONS_DIR = '.workflow/questions'
IGNORE_PATTERNS = [
    '.workflow/',
    '__pycache__',
    '.git/',
    '.pyc',
    '.pytest_cache',
    'node_modules/',
    '.DS_Store',
]


class WatchMode:
    def __init__(self, project_root: str, io=None):
        self.project_root = Path(project_root).resolve()
        self.memory = ProjectMemory(project_root)
        self.tasks = TaskManager(project_root)
        self.io = io
        self._questions_dir = self.project_root / QUESTIONS_DIR
        self._stop_event = asyncio.Event()

    async def start(self) -> None:
        """Démarrer la surveillance — tourne jusqu'à stop()"""
        self._questions_dir.mkdir(parents=True, exist_ok=True)
        if self.io:
            self.io.print_info(f'WatchMode actif sur {self.project_root}')

        async for changes in awatch(str(self.project_root), stop_event=self._stop_event):
            for change_type, path in changes:
                if change_type in (Change.added, Change.modified):
                    await self._handle_change(path)

    def stop(self) -> None:
        self._stop_event.set()

    async def process_answers(self) -> list[dict]:
        """
        Lire les questions répondues et retourner les actions à effectuer.
        Appelé par PhaseManager.run_current_phase() au début de chaque cycle.
        """
        results = []
        if not self._questions_dir.exists():
            return results

        for q_file in sorted(self._questions_dir.glob('QUESTION-*.md')):
            content = q_file.read_text(encoding='utf-8')
            if 'status: pending' not in content:
                continue

            answer_match = re.search(r'\*\*Réponse\*\*\s*:\s*(.+)', content)
            if not answer_match:
                continue

            answer = answer_match.group(1).strip()
            if not answer or answer == '(édite cette ligne)':
                continue

            # La question a été répondue — mettre à jour le statut
            updated = content.replace('status: pending', 'status: answered')
            q_file.write_text(updated, encoding='utf-8')

            file_match = re.search(r'\*\*Fichier modifié\*\*\s*:\s*`(.+?)`', content)
            results.append({
                'question_file': str(q_file),
                'modified_file': file_match.group(1) if file_match else '',
                'answer': answer,
            })

        return results

    async def _handle_change(self, path: str) -> None:
        """Évaluer si le changement est hors scope et créer une question si nécessaire"""
        rel_path = Path(path).relative_to(self.project_root)
        rel_str = str(rel_path)

        # Ignorer les fichiers système/build
        if any(pattern in rel_str for pattern in IGNORE_PATTERNS):
            return

        # Récupérer la tâche courante
        project = await self.memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            return

        progress = await self.memory.get_progress(version)
        in_progress = (progress or {}).get('in_progress', [])
        if not in_progress:
            return

        current_task_id = in_progress[0]
        task = await self.tasks.get_task(version, current_task_id)
        if not task:
            return

        # Vérifier si le fichier est dans le scope de la tâche
        task_files = [
            f.get('path', '') if isinstance(f, dict) else str(f)
            for f in (task.get('files') or task.get('files_to_modify') or [])
        ]
        if any(rel_str.endswith(tf) or tf in rel_str for tf in task_files):
            return  # Dans le scope — pas de question

        # Hors scope → créer une question
        await self._create_question(rel_str, current_task_id, task.get('title', ''))

    async def _create_question(
        self, modified_file: str, task_id: str, task_title: str
    ) -> None:
        """Créer un fichier question dans .workflow/questions/"""
        existing = list(self._questions_dir.glob('QUESTION-*.md'))
        next_num = len(existing) + 1
        q_id = f'QUESTION-{next_num:03d}'
        q_path = self._questions_dir / f'{q_id}.md'

        timestamp = datetime.now(timezone.utc).isoformat()
        content = f"""# {q_id}

**Fichier modifié** : `{modified_file}`
**Hors scope de** : {task_id} ({task_title})

Cette modification semble hors scope de la tâche en cours.
Que veux-tu faire ?

- [ ] Créer une nouvelle tâche pour ce changement
- [ ] Inclure dans {task_id}
- [ ] Ignorer (modification manuelle intentionnelle)

**Réponse** : (édite cette ligne)
---
status: pending
created_at: {timestamp}
"""
        q_path.write_text(content, encoding='utf-8')

        if self.io:
            self.io.print_info(f'Question créée : {q_id} ({modified_file} hors scope)')
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `start()` utilise `watchfiles.awatch` avec `stop_event` | ⬜ |
| 2 | Les fichiers dans `.workflow/`, `__pycache__`, `.git/` sont ignorés | ⬜ |
| 3 | Un fichier modifié **dans** le scope de la tâche ne génère pas de question | ⬜ |
| 4 | Un fichier modifié **hors** scope génère `QUESTION-XXX.md` dans `.workflow/questions/` | ⬜ |
| 5 | Le fichier question contient le fichier modifié, l'ID de tâche, et 3 choix | ⬜ |
| 6 | `process_answers()` lit les questions `status: pending` avec une réponse | ⬜ |
| 7 | `process_answers()` met à jour le statut à `answered` après lecture | ⬜ |
| 8 | `stop()` arrête proprement la surveillance via `_stop_event.set()` | ⬜ |
| 9 | `_create_question()` incrémente le numéro de question sans collision | ⬜ |
