# Phase 1 — Tâche 1.0 : Protocole `.workflow/`

## Objectif

Définir et formaliser `.workflow/` comme un **protocole versionné** — pas un simple dossier de fichiers ad-hoc. Tous les modules Workflow et tous les clients tiers (VS Code extension, Telegram bot, REST API, web dashboard) lisent et écrivent ce protocole via des contrats explicites.

> **Pilier load-bearing #1.** Sans cette discipline, on construit N intégrations qui finissent par diverger dans leur lecture/écriture des fichiers projet.

## Pourquoi un protocole, pas juste des fichiers

| Approche "dossier ad-hoc" | Approche "protocole formalisé" |
|---|---|
| Chaque client réimplémente la logique | Un seul contrat lu par tous |
| Schéma change → casse silencieuse des clients | `schema_version` + migrations automatiques |
| Pas de validation au boot | JSON Schema validés par `SyncChecker` |
| Spec implicite (à reverse-engineer) | Spec publique, doc dédiée |
| Évolution chaotique | Évolution gouvernée (deprecations, compat ascendante) |

## Dépendances

Aucune — premier module de Phase 1.

## Fichiers à Créer

- `src/workflow/protocol/__init__.py` [CRÉER]
- `src/workflow/protocol/schemas/` [CRÉER — JSON Schemas]
  - `project.schema.json`
  - `vision.schema.json`
  - `features.schema.json`
  - `tech-stack.schema.json`
  - `design.schema.json`
  - `version-meta.schema.json`
  - `progress.schema.json`
  - `decisions-graph.schema.json`
  - `failure-patterns.schema.json`
  - `task.schema.json`
- `src/workflow/protocol/version.py` [CRÉER — `SCHEMA_VERSION`, migrations]
- `src/workflow/protocol/validator.py` [CRÉER — validation JSON Schema]
- `src/workflow/protocol/migrations/` [CRÉER]
  - `m_001_initial.py`
- `docs/PROTOCOL.md` [CRÉER — spec publique pour les implémenteurs tiers]
- `tests/unit/test_protocol_validator.py` [CRÉER]
- `tests/unit/test_protocol_migrations.py` [CRÉER]

## Spec du Protocole — `SCHEMA_VERSION`

```python
# src/workflow/protocol/version.py

SCHEMA_VERSION = "1.0.0"

# Politique de versioning :
# MAJEUR : changement breaking (migration obligatoire)
# MINEUR : ajout de champs optionnels (compat ascendante)
# PATCH  : correction de doc/spec, pas de changement de structure
```

Chaque fichier de `.workflow/` qui contient un objet structuré inclut son `schema_version` :

```json
{
  "schema_version": "1.0.0",
  "name": "MyProject",
  "active_version": "v1.0",
  "status": "ACTIVE"
}
```

Les fichiers Markdown (`vision.md`, `TASK-XXX.md`) ne portent pas de `schema_version` mais sont parsés selon une spec stable documentée dans `PROTOCOL.md`.

## Validation au Boot

```python
# src/workflow/protocol/validator.py
import json
from pathlib import Path
from jsonschema import validate, ValidationError

SCHEMAS_DIR = Path(__file__).parent / 'schemas'


class ProtocolValidator:
    def __init__(self):
        self._schemas = {}
        for schema_file in SCHEMAS_DIR.glob('*.schema.json'):
            name = schema_file.stem.replace('.schema', '')
            self._schemas[name] = json.loads(schema_file.read_text())

    def validate(self, schema_name: str, data: dict) -> tuple[bool, str | None]:
        """Retourne (True, None) ou (False, error_message)"""
        schema = self._schemas.get(schema_name)
        if not schema:
            return False, f'Schema inconnu : {schema_name}'
        try:
            validate(instance=data, schema=schema)
            return True, None
        except ValidationError as e:
            return False, f'{e.message} (chemin: {".".join(str(p) for p in e.absolute_path)})'

    def validate_workflow_dir(self, workflow_root: Path) -> list[dict]:
        """
        Valider tous les fichiers JSON d'un dossier .workflow/.
        Retourne la liste des erreurs (vide si tout OK).
        """
        errors = []
        mappings = {
            'project.json': 'project',
            'features.json': 'features',
            'tech-stack.json': 'tech-stack',
            'design.json': 'design',
            'decisions-graph.json': 'decisions-graph',
            'failure-patterns.json': 'failure-patterns',
        }
        for filename, schema_name in mappings.items():
            filepath = workflow_root / filename
            if not filepath.exists():
                continue
            try:
                data = json.loads(filepath.read_text(encoding='utf-8'))
            except json.JSONDecodeError as e:
                errors.append({'file': filename, 'error': f'JSON invalide : {e}'})
                continue
            ok, msg = self.validate(schema_name, data)
            if not ok:
                errors.append({'file': filename, 'error': msg})
        return errors
```

## Migrations Automatiques

```python
# src/workflow/protocol/migrations/__init__.py
from typing import Callable

# Registre des migrations : version source → fonction de migration
MIGRATIONS: dict[str, Callable[[dict], dict]] = {}


def register_migration(from_version: str):
    def decorator(func):
        MIGRATIONS[from_version] = func
        return func
    return decorator


def migrate_to_current(data: dict, current_version: str) -> dict:
    """Appliquer toutes les migrations nécessaires pour amener data à current_version"""
    while data.get('schema_version', '0.0.0') != current_version:
        from_v = data.get('schema_version', '0.0.0')
        migration = MIGRATIONS.get(from_v)
        if not migration:
            raise ValueError(f'Pas de migration depuis {from_v} vers {current_version}')
        data = migration(data)
    return data
```

Exemple de migration future (v1.0.0 → v1.1.0) :

```python
# src/workflow/protocol/migrations/m_001_v1_0_to_v1_1.py
from . import register_migration


@register_migration('1.0.0')
def migrate(data: dict) -> dict:
    """v1.0.0 → v1.1.0 : ajout du champ 'tags' (array, default [])"""
    data['tags'] = data.get('tags', [])
    data['schema_version'] = '1.1.0'
    return data
```

## Spec Publique — `docs/PROTOCOL.md`

Le fichier `docs/PROTOCOL.md` est la **spec lisible par les implémenteurs tiers**. Il documente :

1. La structure complète de `.workflow/`
2. Le format de chaque fichier (JSON Schema + exemples)
3. La spec des fichiers Markdown (TASK-XXX.md, vision.md)
4. Les invariants (ex : "exactement une version est ACTIVE à la fois")
5. Les opérations atomiques requises (write-then-rename pour éviter la corruption)
6. La politique de versioning (`SCHEMA_VERSION` + migrations)
7. Les conventions d'extension (`x-*` pour les champs custom des clients tiers)

Cette spec sera versionnée dans le repo public — un développeur tiers peut écrire un client (web app, mobile app) compatible sans connaître l'implémentation Python.

## Schéma Exemple — `project.schema.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://workflow.dev/schemas/v1/project.schema.json",
  "type": "object",
  "required": ["schema_version", "name", "created_at", "status", "active_contexts"],
  "properties": {
    "schema_version": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "name": { "type": "string", "minLength": 1, "maxLength": 100 },
    "description": { "type": "string", "maxLength": 500 },
    "created_at": { "type": "string", "format": "date-time" },
    "active_version": { "type": ["string", "null"] },
    "status": {
      "enum": ["DISCOVERY", "SPECIFICATION", "DESIGN", "ARCHITECTURE", "VALIDATION", "ACTIVE"]
    },
    "active_contexts": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1,
      "description": "Contexts spécialisés actifs pour ce projet (Pilier 7). Au minimum [_global]. Ordonnés du plus général au plus spécifique."
    },
    "phase_history": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "from": { "type": "string" },
          "to": { "type": "string" },
          "kind": { "enum": ["FORWARD", "REVISIT", "STARTER_JUMP"] },
          "reason": { "type": "string" },
          "timestamp": { "type": "string", "format": "date-time" }
        }
      }
    }
  },
  "additionalProperties": false
}
```

> Le champ `active_contexts` est requis. Au boot d'un projet sans contexts définis, `SyncChecker` corrige en y ajoutant `["_global"]`. Voir `phase-1-foundation/09-contexts.md`.

## Intégration avec les autres modules

**`ProjectMemory` (tâche 1.3)** — utilise le protocole pour valider à l'écriture et à la lecture :

```python
async def update_project(self, data: dict):
    ok, msg = self.validator.validate('project', data)
    if not ok:
        raise ValueError(f'project.json invalide : {msg}')
    await self.fs.write_json('project.json', data)
```

**`SyncChecker` (tâche 1.8)** — au boot, valide tout `.workflow/` et applique les migrations :

```python
async def check(self) -> dict:
    errors = self.validator.validate_workflow_dir(self.workflow_root)
    if errors:
        return {'ok': False, 'issues': [f'{e["file"]} : {e["error"]}' for e in errors]}

    # Vérifier schema_version + migrer si besoin
    project = await self.memory.get_project()
    if project.get('schema_version') != SCHEMA_VERSION:
        migrated = migrate_to_current(project, SCHEMA_VERSION)
        await self.memory.update_project(migrated)
    ...
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `SCHEMA_VERSION` est exposée et utilisée par tous les modules d'écriture | ⬜ |
| 2 | `ProtocolValidator.validate('project', data)` retourne `(True, None)` sur un projet valide | ⬜ |
| 3 | `validate('project', data)` retourne `(False, msg)` avec un message clair sur un projet invalide | ⬜ |
| 4 | `validate_workflow_dir()` détecte un JSON corrompu et retourne une erreur lisible | ⬜ |
| 5 | `migrate_to_current()` applique successivement toutes les migrations nécessaires | ⬜ |
| 6 | `migrate_to_current()` lève `ValueError` si une migration manque | ⬜ |
| 7 | Tous les schémas JSON sont valides selon Draft 2020-12 | ⬜ |
| 8 | `docs/PROTOCOL.md` couvre la totalité des fichiers `.workflow/` | ⬜ |
| 9 | Tests unitaires couvrent : validation OK, validation KO, migration v1.0→v1.1 (mock) | ⬜ |
| 10 | Les schémas autorisent l'extension `x-*` pour les clients tiers | ⬜ |

## Notes d'Implémentation

- **Atomicité** : tous les writes JSON passent par `FileSystem.atomic_write()` (write-temp + rename) pour éviter la corruption en cas de crash mid-write.
- **Retro-compat** : un client qui lit un fichier avec un `schema_version` plus récent doit afficher un avertissement explicite, pas crasher silencieusement.
- **Champs custom** : les clients tiers peuvent ajouter des champs `x-vendor-key` sans casser la validation (`additionalProperties: false` ne s'applique qu'aux champs sans préfixe `x-`).
