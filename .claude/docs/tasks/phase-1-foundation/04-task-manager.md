# Phase 1 — Tâche 1.4 : TaskManager.py

## Objectif

Créer le module `TaskManager.py` qui gère le CRUD sur les fichiers `TASK-XXX.md` et les fichiers `progress.json`. C'est ce module qui impose le format auto-suffisant des tâches et garantit la numérotation séquentielle.

## Dépendances

- Tâche 1.2 ✅ (`FileSystem.py`)
- Tâche 1.3 ✅ (`ProjectMemory.py`)

## Fichiers à Créer / Modifier

- `src/workflow/tools/task_manager.py` [CRÉER]
- `tests/unit/test_task_manager.py` [CRÉER]

## Format des Fichiers de Tâche

Voir `CLAUDE.md#format-des-tâches-auto-suffisantes` pour le format complet.

**Statuts possibles** : `⬜ EN ATTENTE` | `🔄 EN COURS` | `✅ TERMINÉ` | `❌ REPORTÉ`

## Implémentation

```python
# src/workflow/tools/task_manager.py
import re
from datetime import date as date_cls
from workflow.tools.filesystem import FileSystem
from workflow.core.project_memory import ProjectMemory


class TaskManager:
    def __init__(self, project_root: str):
        self.fs = FileSystem(project_root)
        self.memory = ProjectMemory(project_root)

    async def next_task_id(self, version: str) -> str:
        """Générer le prochain ID de tâche (TASK-001, TASK-002...)"""
        existing = await self.fs.list_task_ids(version)
        if not existing:
            return 'TASK-001'
        last = existing[-1]
        num = int(last.replace('TASK-', ''))
        return f'TASK-{num + 1:03d}'

    async def create_task(self, version: str, task_data: dict) -> str:
        """Créer une nouvelle tâche"""
        task_id = task_data.get('id') or await self.next_task_id(version)
        content = self.render_task_file({
            **task_data,
            'id': task_id,
            'version': version,
            'status': '⬜ EN ATTENTE',
        })
        await self.fs.write_markdown(self.fs.paths.task_file(version, task_id), content)

        # Ajouter dans progress.json
        progress = await self.memory.get_progress(version)
        progress['pending'].append(task_id)
        await self.memory.save_progress(version, progress)
        return task_id

    async def get_task(self, version: str, task_id: str) -> dict | None:
        """Lire une tâche (parser le Markdown → objet)"""
        content = await self.fs.read_markdown(self.fs.paths.task_file(version, task_id))
        if not content:
            return None
        return self.parse_task_file(task_id, version, content)

    async def mark_done(self, version: str, task_id: str):
        """Marquer une tâche comme terminée"""
        await self._update_task_status(version, task_id, '✅ TERMINÉ')
        progress = await self.memory.get_progress(version)
        progress['pending'] = [t for t in progress['pending'] if t != task_id]
        if task_id not in progress['done']:
            progress['done'].append(task_id)
        await self.memory.save_progress(version, progress)

    async def mark_in_progress(self, version: str, task_id: str):
        """Marquer une tâche comme en cours"""
        await self._update_task_status(version, task_id, '🔄 EN COURS')

    async def defer_task(self, version: str, task_id: str, target_version: str, reason: str):
        """Reporter une tâche à une autre version"""
        await self._update_task_status(version, task_id, '❌ REPORTÉ')
        progress = await self.memory.get_progress(version)
        progress['pending'] = [t for t in progress['pending'] if t != task_id]
        progress['deferred'].append({'id': task_id, 'to': target_version, 'reason': reason})
        await self.memory.save_progress(version, progress)
        await self.append_journal(version, task_id, f'Reportée vers {target_version} — raison : {reason}')

    async def append_journal(self, version: str, task_id: str, entry: str):
        """Ajouter une entrée dans le champ Journal de la tâche"""
        content = await self.fs.read_markdown(self.fs.paths.task_file(version, task_id))
        if not content:
            return
        today = date_cls.today().isoformat()

        if '(vide — tâche jamais tentée)' in content:
            updated = content.replace(
                '(vide — tâche jamais tentée)',
                f'[{today}] {entry}'
            )
        else:
            updated = re.sub(
                r'(\n{1,2}## Statut)',
                f'\n[{today}] {entry}\n\\1',
                content
            )
        await self.fs.write_markdown(self.fs.paths.task_file(version, task_id), updated)

    async def get_pending_tasks(self, version: str) -> list[str]:
        """Lister les tâches en attente d'une version"""
        progress = await self.memory.get_progress(version)
        return progress['pending']

    async def get_next_task(self, version: str) -> dict | None:
        """Prochaine tâche à exécuter (première EN ATTENTE)"""
        pending = await self.get_pending_tasks(version)
        if not pending:
            return None
        return await self.get_task(version, pending[0])

    async def _update_task_status(self, version: str, task_id: str, status: str):
        """Mettre à jour le statut dans le fichier Markdown"""
        content = await self.fs.read_markdown(self.fs.paths.task_file(version, task_id))
        if not content:
            return
        updated = re.sub(r'## Statut\n.*', f'## Statut\n{status}', content)
        await self.fs.write_markdown(self.fs.paths.task_file(version, task_id), updated)

    def render_task_file(self, task: dict) -> str:
        """Rendre un objet tâche en Markdown"""
        deps = '\n'.join(
            f"- {d['id']} {'✅' if d.get('done') else '⬜'} ({d.get('description', '')})"
            for d in (task.get('dependencies') or [])
        ) or '(aucune)'

        files = '\n'.join(
            f"- {f['path']} [{f['action']}]"
            for f in (task.get('files') or [])
        ) or '(aucun)'

        criteria = '\n'.join(
            f'- [ ] {c}'
            for c in (task.get('criteria') or [])
        ) or '- [ ] À définir'

        preconditions_raw = task.get('preconditions') or {}
        if preconditions_raw:
            import json
            preconditions = '\n'.join(
                f'- {k}: {json.dumps(v)}'
                for k, v in preconditions_raw.items()
            )
        else:
            preconditions = '(aucune)'

        # Rendu du mockup UI
        mockup = task.get('mockup')
        if mockup and mockup.get('screens'):
            mockup_section = '\n\n'.join(
                f"### Écran — {s['name']}\n{s['ascii']}\n"
                + (f"Style : {s['notes']}" if s.get('notes') else '')
                for s in mockup['screens']
            )
        else:
            mockup_section = '(aucune interface — tâche backend / configuration)'

        return f"""# {task['id']} : {task.get('title', '')}
## Version : {task.get('version', '')}

## Contexte Projet
{task.get('context') or '(à compléter)'}

## User Story
{task.get('user_story') or task.get('userStory') or '(à compléter)'}

## Intent
{task.get('intent') or "(à compléter — pourquoi l'utilisateur veut vraiment cette fonctionnalité)"}

## Préconditions
{preconditions}

## Dépendances
{deps}

## Fichiers à créer / modifier
{files}

## Critères d'acceptation
{criteria}

## Mockup UI
{mockup_section}

## Journal
(vide — tâche jamais tentée)

## Statut
{task.get('status', '⬜ EN ATTENTE')}
"""

    def parse_task_file(self, task_id: str, version: str, content: str) -> dict:
        """Parser un fichier Markdown de tâche → objet"""
        def get_section(name: str) -> str:
            match = re.search(rf'## {re.escape(name)}\n([\s\S]*?)(?=\n## |$)', content)
            return match.group(1).strip() if match else ''

        # Parser les préconditions
        preconditions_raw = get_section('Préconditions')
        preconditions: dict | None = None
        if preconditions_raw and preconditions_raw != '(aucune)':
            import json
            preconditions = {}
            for line in preconditions_raw.split('\n'):
                if line.startswith('- '):
                    key, _, rest = line[2:].partition(':')
                    try:
                        preconditions[key.strip()] = json.loads(rest.strip())
                    except (json.JSONDecodeError, ValueError):
                        preconditions[key.strip()] = rest.strip()

        # Parser les mockups
        mockup_raw = get_section('Mockup UI')
        mockup: dict | None = None
        if mockup_raw and not mockup_raw.startswith('(aucune'):
            screens = []
            for block in re.split(r'\n(?=### Écran — )', mockup_raw):
                name_match = re.match(r'^### Écran — (.+)', block)
                if not name_match:
                    continue
                name = name_match.group(1).strip()
                rest = block[block.index('\n') + 1:]
                style_match = re.search(r'\nStyle : (.+)$', rest, re.MULTILINE)
                notes = style_match.group(1).strip() if style_match else None
                ascii_art = rest[:rest.rfind('\nStyle :')] if style_match else rest
                screens.append({'name': name, 'ascii': ascii_art.strip(), 'notes': notes})
            if screens:
                mockup = {'screens': screens}

        title_match = re.match(r'^# \S+ : (.+)', content, re.MULTILINE)

        return {
            'id': task_id,
            'version': version,
            'title': title_match.group(1) if title_match else '',
            'context': get_section('Contexte Projet'),
            'user_story': get_section('User Story'),
            'intent': get_section('Intent'),
            'preconditions': preconditions,
            'files_to_modify': [
                line[2:].split(' [')[0]
                for line in get_section('Fichiers à créer / modifier').split('\n')
                if line.startswith('- ')
            ],
            'criteria': [
                re.sub(r'^- \[.\] ', '', line)
                for line in get_section("Critères d'acceptation").split('\n')
                if line.startswith('- ')
            ],
            'mockup': mockup,
            'journal': get_section('Journal'),
            'status': get_section('Statut'),
        }
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `next_task_id` génère TASK-001, TASK-002... séquentiellement | ⬜ |
| 2 | `create_task` écrit le fichier Markdown ET met à jour `progress.json` | ⬜ |
| 3 | `mark_done` déplace l'ID de `pending` vers `done` | ⬜ |
| 4 | `defer_task` ajoute une entrée dans le champ Journal du fichier | ⬜ |
| 4b | `append_journal` remplace `(vide — tâche jamais tentée)` à la première entrée | ⬜ |
| 4c | `append_journal` appende correctement une 2ème entrée sous la première | ⬜ |
| 5 | `parse_task_file` extrait correctement toutes les sections dont `intent` et `preconditions` | ⬜ |
| 6 | `render_task_file` inclut les sections `## Intent`, `## Préconditions`, `## Mockup UI` | ⬜ |
| 7 | `parse_task_file` retourne `preconditions: None` si la section est absente ou vide | ⬜ |
| 8 | `parse_task_file` retourne `mockup: None` si la section commence par `(aucune` | ⬜ |
| 9 | Tests unitaires couvrent le cycle complet d'une tâche | ⬜ |
