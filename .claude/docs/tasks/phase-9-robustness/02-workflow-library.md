# Phase 6 — Tâche 6.4 : WorkflowLibrary.py

## Objectif

Créer la `WorkflowLibrary` — une mémoire locale **cross-projet** qui capitalise les patterns validés et les erreurs connues à travers tous les projets gérés par Workflow. Plus Workflow est utilisé, plus il devient précis sur les types de projets que l'utilisateur construit.

> **Note** : `SkillManager` (Phase 1, tâche 1.0) gère les skills bas-niveau (fixes de retry).
> `WorkflowLibrary` gère les patterns haut-niveau (tâches entières pré-remplies, patterns d'architecture).
> Les deux sont complémentaires.

## Dépendances

- Phase 4 (MVP) ✅
- Tâche 6.2 (`CodeIndexer`) recommandée ✅

## Fichiers à Créer

- `src/workflow/tools/workflow_library.py` [CRÉER]
- `tests/unit/test_workflow_library.py` [CRÉER]

## Structure de la Librairie

Stockée dans `~/.workflow/library/` (global, hors des projets) :

```
~/.workflow/library/
├── patterns/
│   ├── jwt-auth-fastapi.md     # Pattern validé sur N projets
│   ├── prisma-setup.md
│   ├── pytest-fixtures.md
│   └── ...
├── failure-patterns.json       # Erreurs connues cross-projet
└── index.json                  # Index des patterns avec tags et stats
```

## Format d'un Pattern

```markdown
---
id: jwt-auth-fastapi
title: Authentification JWT — FastAPI
tags: [auth, jwt, fastapi, python]
used_in: 3
success_rate: 1.0
last_used: 2026-04-05
---

## Stack
Python + FastAPI + python-jose + passlib

## Files
- src/middleware/auth.py
- src/services/jwt_service.py
- src/routes/auth_routes.py

## Critères qui déclenchent ce pattern
- Tâche contient "auth" + "JWT" dans title ou intent
- Stack = Python + FastAPI

## Template de tâche pré-rempli
[contenu Markdown complet de la TASK pré-remplie]

## Notes
Fonctionne sans modification sur FastAPI 0.100+.
```

## Implémentation

```python
# src/workflow/tools/workflow_library.py
import json
import re
from datetime import datetime, timezone
from pathlib import Path

LIBRARY_DIR = Path.home() / '.workflow' / 'library'


class WorkflowLibrary:
    def __init__(self):
        self.patterns_dir = LIBRARY_DIR / 'patterns'
        self.failure_patterns_path = LIBRARY_DIR / 'failure-patterns.json'
        self.index_path = LIBRARY_DIR / 'index.json'

    def init(self) -> None:
        """Initialiser la librairie globale (première utilisation)"""
        self.patterns_dir.mkdir(parents=True, exist_ok=True)
        if not self.index_path.exists():
            self.index_path.write_text('[]', encoding='utf-8')
        if not self.failure_patterns_path.exists():
            self.failure_patterns_path.write_text('[]', encoding='utf-8')

    def find_pattern(self, task: dict) -> dict | None:
        """Chercher un pattern applicable à une tâche"""
        index = self._read_index()
        search_text = ' '.join(filter(None, [
            task.get('title', ''),
            task.get('intent', ''),
            task.get('context', ''),
        ])).lower()

        candidates = sorted(
            [{'score': self._score_pattern(e, search_text), **e} for e in index],
            key=lambda x: x['score'],
            reverse=True,
        )
        best = next((c for c in candidates if c['score'] > 0.5), None)
        if not best:
            return None

        pattern_file = self.patterns_dir / f"{best['id']}.md"
        if not pattern_file.exists():
            return None

        return {**best, 'content': pattern_file.read_text(encoding='utf-8')}

    def save_pattern(self, task: dict) -> None:
        """Enregistrer un pattern depuis une tâche réussie"""
        index = self._read_index()
        pattern_id = self._generate_id(task.get('title', ''))
        existing = next((e for e in index if e['id'] == pattern_id), None)

        today = datetime.now(timezone.utc).date().isoformat()
        if existing:
            existing['used_in'] = existing.get('used_in', 1) + 1
            existing['last_used'] = today
        else:
            tags = self._extract_tags(task)
            index.append({
                'id': pattern_id,
                'title': task.get('title', ''),
                'tags': tags,
                'used_in': 1,
                'success_rate': 1.0,
                'last_used': today,
            })
            pattern_content = self._render_pattern(task, pattern_id, tags, today)
            (self.patterns_dir / f'{pattern_id}.md').write_text(pattern_content, encoding='utf-8')

        self.index_path.write_text(
            json.dumps(index, indent=2, ensure_ascii=False), encoding='utf-8'
        )

    def merge_project_failures(self, project_failure_path: Path) -> None:
        """Fusionner les failure-patterns d'un projet avec la librairie globale"""
        try:
            project_patterns = json.loads(project_failure_path.read_text(encoding='utf-8'))
        except (FileNotFoundError, json.JSONDecodeError):
            return

        global_patterns = self._read_failure_patterns()
        for pp in project_patterns:
            existing = next(
                (g for g in global_patterns if g.get('fingerprint') == pp.get('fingerprint')), None
            )
            if existing:
                existing['occurrences'] = existing.get('occurrences', 1) + pp.get('occurrences', 1)
                existing['last_seen'] = pp.get('last_seen', existing['last_seen'])
            else:
                global_patterns.append({**pp, 'source': 'project'})

        self.failure_patterns_path.write_text(
            json.dumps(global_patterns, indent=2, ensure_ascii=False), encoding='utf-8'
        )

    def get_global_failure_patterns(self) -> list[dict]:
        return self._read_failure_patterns()

    def _score_pattern(self, entry: dict, search_text: str) -> float:
        tags = ' '.join(entry.get('tags', [])).lower()
        title = entry.get('title', '').lower()
        combined = f'{tags} {title}'
        words = [w for w in re.split(r'\W+', search_text) if len(w) > 3]
        if not words:
            return 0.0
        matches = sum(1 for w in words if w in combined)
        recency_bonus = 0.1 if entry.get('used_in', 0) > 2 else 0.0
        return (matches / len(words)) + recency_bonus

    def _generate_id(self, title: str) -> str:
        return re.sub(r'[^a-z0-9]+', '-', title.lower())[:40].strip('-')

    def _extract_tags(self, task: dict) -> list[str]:
        technical_terms = [
            'auth', 'jwt', 'prisma', 'postgres', 'redis', 'api', 'rest',
            'graphql', 'pytest', 'ruff', 'fastapi', 'django', 'sqlalchemy',
        ]
        text = f"{task.get('title', '')} {task.get('context', '')}".lower()
        return [t for t in technical_terms if t in text]

    def _render_pattern(
        self, task: dict, pattern_id: str, tags: list[str], today: str
    ) -> str:
        files = '\n'.join(
            f"- {f['path']}" if isinstance(f, dict) else f'- {f}'
            for f in (task.get('files') or [])
        ) or '(aucun)'
        criteria = '\n'.join(
            f'- {c}' for c in (task.get('criteria') or [])
        ) or '(aucun)'

        return f"""---
id: {pattern_id}
title: {task.get('title', '')}
tags: [{', '.join(tags)}]
used_in: 1
success_rate: 1.0
last_used: {today}
---

## Files
{files}

## Critères qui déclenchent ce pattern
- Tâche contient "{tags[0] if tags else task.get('title', '').split()[0]}" dans title ou intent

## Template de tâche pré-rempli
### Title
{task.get('title', '')}

### Intent
{task.get('intent') or '(à compléter)'}

### Criteria
{criteria}

## Notes
Pattern extrait automatiquement depuis une tâche réussie.
"""

    def _read_index(self) -> list[dict]:
        try:
            return json.loads(self.index_path.read_text(encoding='utf-8'))
        except (FileNotFoundError, json.JSONDecodeError):
            return []

    def _read_failure_patterns(self) -> list[dict]:
        try:
            return json.loads(self.failure_patterns_path.read_text(encoding='utf-8'))
        except (FileNotFoundError, json.JSONDecodeError):
            return []
```

## Intégration dans le Workflow Existant

**Au démarrage d'un projet** (`workflow init`) :
```python
library = WorkflowLibrary()
library.init()
global_failures = library.get_global_failure_patterns()
# Pré-charger dans le projet local
```

**Lors de la génération d'une tâche** (`ValidationPhase`) :
```python
pattern = library.find_pattern(task)
if pattern:
    io.print_info(f'Pattern trouvé : "{pattern["title"]}" (utilisé sur {pattern["used_in"]} projets)')
    if io.confirm('Pré-remplir la tâche avec ce pattern ?'):
        task = apply_pattern(task, pattern)
```

**Après complétion d'une version** (`workflow version complete`) :
```python
library.merge_project_failures(Path(project_root) / '.workflow' / 'failure-patterns.json')
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `init()` crée `~/.workflow/library/` si absent | ⬜ |
| 2 | `find_pattern()` retourne `None` si aucun pattern avec score > 0.5 | ⬜ |
| 3 | `find_pattern()` retourne le meilleur pattern si score suffisant | ⬜ |
| 4 | `save_pattern()` incrémente `used_in` si le pattern existe déjà | ⬜ |
| 5 | `merge_project_failures()` agrège les occurrences sans dupliquer | ⬜ |
| 6 | `get_global_failure_patterns()` retourne `[]` si fichier absent | ⬜ |
| 7 | Tests unitaires utilisent un répertoire temporaire (pas `~/.workflow/library/` réel) | ⬜ |
| 8 | `save_pattern()` écrit un fichier Markdown valide dans `patterns/` | ⬜ |
