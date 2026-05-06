# Phase 1 — Tâche 1.9 : Système de Contexts

## Objectif

Créer le **système de contexts** — la couche de spécialisation qui transforme Workflow d'un agent à mémoire unique en **cabinet de devs spécialisés** (Pilier 7). Un context est un namespace hiérarchique avec ses propres skills, decisions, defaults stack/design, et règles d'apprentissage.

> **Pilier load-bearing #7.** Sans contexts, après 10 projets mixtes (mobile + web + backend), les patterns se polluent — les skills `auth` Flutter polluent les projets web. Avec contexts, l'agent devient *spécialisé par domaine*.

## Concept Clé

```
~/.workflow/contexts/
└── _global/                        ← le seul context au moment de pip install
    ├── config.yaml
    └── skills/                      (vide)

# Après usage :
~/.workflow/contexts/
├── _global/                        [parent: aucun]
│   ├── config.yaml
│   ├── skills/                     ← patterns universels (commits, sécurité)
│   ├── decisions.log               ← "toujours hash bcrypt", etc.
│   └── teach.md                    ← USER_OVERRIDE rules globales
├── mobile/                         [parent: _global]
│   ├── config.yaml
│   ├── tech-stack-defaults.json    ← stacks proposées par défaut
│   ├── design-defaults.json        ← Material/Cupertino par défaut
│   ├── skills/                     ← patterns mobile-general
│   └── decisions.log
├── mobile.flutter/                 [parent: mobile]
│   ├── config.yaml
│   └── skills/                     ← patterns Flutter-only (BLoC, go_router)
├── web/                            [parent: _global]
└── web.nextjs/                     [parent: web]
```

## Templates Bundled

Workflow embarque **12 templates de contexts** dans son package (pas dans `~/.workflow/`). Ce sont des **scaffolds opinionés** — config + defaults, **pas de skills** :

```
src/workflow/templates/contexts/
├── mobile/
├── mobile.flutter/
├── mobile.react-native/
├── web/
├── web.nextjs/
├── web.vite-react/
├── desktop/
├── desktop.electron/
├── desktop.tauri/
├── backend/
├── backend.fastapi/
└── backend.express/
```

Au `workflow init`, si un context approprié n'existe pas chez l'utilisateur, Workflow propose de l'**instancier depuis le template** (copie config + defaults dans `~/.workflow/contexts/`).

## Dépendances

- Tâche 1.0 ✅ (`Protocol` — schéma `project.json` doit inclure `active_contexts`)
- Tâche 1.2 ✅ (`FileSystem`)

## Fichiers à Créer

- `src/workflow/core/context_registry.py` [CRÉER]
- `src/workflow/templates/contexts/` [CRÉER — 12 templates avec config + defaults]
- `src/workflow/protocol/schemas/context.schema.json` [CRÉER]
- `tests/unit/test_context_registry.py` [CRÉER]

## Schéma `~/.workflow/contexts/<name>/config.yaml`

```yaml
schema_version: "1.0.0"
name: "mobile.flutter"
parent: "mobile"
description: "Apps mobile cross-platform avec Flutter / Dart"
created_at: "2026-04-12T10:00:00Z"
created_from_template: true
template_version: "1.0.0"

# Routage LLM spécifique au context (optionnel — sinon hérite du parent)
models:
  code_generation: "deepseek/deepseek-coder-v2"   # itération rapide widgets
  reasoning: "anthropic/claude-opus-4-7"

# Tags utilisés pour le matching de skills/décisions
tags: ["mobile", "flutter", "dart", "cross-platform"]

# Heuristique d'auto-détection
detection:
  files: ["pubspec.yaml"]
  required_in_pubspec: ["flutter:"]

# Conventions par défaut
conventions:
  state_management: "BLoC (proposé, peut être surchargé)"
  navigation: "go_router"
  testing: "flutter_test + mocktail"
```

## Schéma `tech-stack-defaults.json`

```json
{
  "schema_version": "1.0.0",
  "language": "dart",
  "framework": "flutter",
  "package_manager": "pub",
  "build_validate": "flutter analyze && dart format --set-exit-if-changed .",
  "test": "flutter test",
  "allowed_commands": [
    "flutter pub get", "flutter pub add", "flutter analyze",
    "flutter test", "dart format", "flutter build"
  ],
  "suggested_dependencies": ["flutter_bloc", "go_router", "dio"]
}
```

## Schéma `project.json` étendu

Ajouter dans `phase-1-foundation/00-protocol.md` schema :

```json
{
  "schema_version": "1.0.0",
  "name": "FoodDelivery",
  "active_contexts": ["_global", "mobile", "mobile.flutter"],
  "...": "..."
}
```

## Implémentation

```python
# src/workflow/core/context_registry.py
import shutil
import yaml
from pathlib import Path
from datetime import datetime, timezone
from dataclasses import dataclass

from workflow.tools.filesystem import FileSystem


CONTEXTS_HOME = Path.home() / '.workflow' / 'contexts'

# Localisé dans le package (resources)
TEMPLATES_DIR = Path(__file__).parent.parent / 'templates' / 'contexts'

GLOBAL_CONTEXT_NAME = '_global'


@dataclass
class ContextInfo:
    name: str
    parent: str | None
    description: str
    skills_count: int
    decisions_count: int
    is_template: bool        # True si seulement dispo en template, pas instancié
    path: Path | None        # None si template seulement


class ContextRegistry:
    """
    Gère la hiérarchie des contexts utilisateur (~/.workflow/contexts/).
    Lazy instantiation depuis les templates bundled.
    """

    def __init__(self):
        self.contexts_home = CONTEXTS_HOME
        self.templates_dir = TEMPLATES_DIR
        self._ensure_global()

    def _ensure_global(self):
        """Garantir que _global existe au boot"""
        global_path = self.contexts_home / GLOBAL_CONTEXT_NAME
        if not global_path.exists():
            global_path.mkdir(parents=True, exist_ok=True)
            (global_path / 'skills').mkdir(exist_ok=True)
            self._write_config(global_path, {
                'schema_version': '1.0.0',
                'name': GLOBAL_CONTEXT_NAME,
                'parent': None,
                'description': 'Universal patterns applicable across all domains',
                'created_at': datetime.now(timezone.utc).isoformat(),
                'created_from_template': False,
            })

    # ─── Listing ──────────────────────────────────────────────────────

    def list_active(self) -> list[ContextInfo]:
        """Lister les contexts effectivement créés chez l'utilisateur"""
        result: list[ContextInfo] = []
        if not self.contexts_home.exists():
            return result

        for ctx_dir in sorted(self.contexts_home.iterdir()):
            if not ctx_dir.is_dir():
                continue
            config = self._read_config(ctx_dir)
            if not config:
                continue
            result.append(ContextInfo(
                name=config['name'],
                parent=config.get('parent'),
                description=config.get('description', ''),
                skills_count=len(list((ctx_dir / 'skills').glob('*.md'))) if (ctx_dir / 'skills').exists() else 0,
                decisions_count=self._count_decisions(ctx_dir),
                is_template=False,
                path=ctx_dir,
            ))
        return result

    def list_available_templates(self) -> list[ContextInfo]:
        """Lister les templates bundled qui ne sont pas encore instanciés"""
        result: list[ContextInfo] = []
        if not self.templates_dir.exists():
            return result

        active_names = {c.name for c in self.list_active()}
        for tpl_dir in sorted(self.templates_dir.iterdir()):
            if not tpl_dir.is_dir():
                continue
            config = self._read_config(tpl_dir)
            if not config or config['name'] in active_names:
                continue
            result.append(ContextInfo(
                name=config['name'],
                parent=config.get('parent'),
                description=config.get('description', ''),
                skills_count=0,
                decisions_count=0,
                is_template=True,
                path=None,
            ))
        return result

    def get(self, name: str) -> ContextInfo | None:
        """Retourner un context actif ou None"""
        ctx_dir = self.contexts_home / name
        if not ctx_dir.exists():
            return None
        config = self._read_config(ctx_dir)
        if not config:
            return None
        return ContextInfo(
            name=config['name'],
            parent=config.get('parent'),
            description=config.get('description', ''),
            skills_count=len(list((ctx_dir / 'skills').glob('*.md'))) if (ctx_dir / 'skills').exists() else 0,
            decisions_count=self._count_decisions(ctx_dir),
            is_template=False,
            path=ctx_dir,
        )

    def get_path(self, name: str) -> Path:
        """Retourner le chemin d'un context (lève si absent)"""
        ctx = self.get(name)
        if not ctx:
            raise ValueError(f'Context absent : {name}')
        return ctx.path

    # ─── Création ─────────────────────────────────────────────────────

    def create_from_template(self, name: str) -> ContextInfo:
        """Instancier un context depuis le template bundled"""
        if self.get(name):
            raise ValueError(f'Context "{name}" existe déjà')

        template_dir = self.templates_dir / name
        if not template_dir.exists():
            raise ValueError(f'Aucun template bundled pour "{name}"')

        # Copier le template vers ~/.workflow/contexts/
        target = self.contexts_home / name
        shutil.copytree(template_dir, target)

        # S'assurer que skills/ existe vide
        (target / 'skills').mkdir(exist_ok=True)

        # Marquer dans la config
        config = self._read_config(target)
        config['created_at'] = datetime.now(timezone.utc).isoformat()
        config['created_from_template'] = True
        self._write_config(target, config)

        # Récursivement créer les parents si nécessaire
        parent_name = config.get('parent')
        if parent_name and parent_name != GLOBAL_CONTEXT_NAME and not self.get(parent_name):
            self.create_from_template(parent_name)

        return self.get(name)

    def create_custom(
        self, name: str, parent: str = GLOBAL_CONTEXT_NAME, description: str = ''
    ) -> ContextInfo:
        """Créer un context custom (non basé sur un template)"""
        if self.get(name):
            raise ValueError(f'Context "{name}" existe déjà')
        if parent and not self.get(parent):
            raise ValueError(f'Parent "{parent}" inexistant')

        target = self.contexts_home / name
        target.mkdir(parents=True, exist_ok=True)
        (target / 'skills').mkdir(exist_ok=True)

        self._write_config(target, {
            'schema_version': '1.0.0',
            'name': name,
            'parent': parent,
            'description': description,
            'created_at': datetime.now(timezone.utc).isoformat(),
            'created_from_template': False,
        })
        return self.get(name)

    def fork(self, source: str, target_name: str) -> ContextInfo:
        """Cloner un context existant (skills + decisions inclus)"""
        src = self.get(source)
        if not src:
            raise ValueError(f'Source inexistante : {source}')
        if self.get(target_name):
            raise ValueError(f'Target existe déjà : {target_name}')

        target = self.contexts_home / target_name
        shutil.copytree(src.path, target)

        # Mettre à jour la config
        config = self._read_config(target)
        config['name'] = target_name
        config['created_at'] = datetime.now(timezone.utc).isoformat()
        config['forked_from'] = source
        self._write_config(target, config)
        return self.get(target_name)

    def delete(self, name: str):
        """Supprimer un context (refuse _global)"""
        if name == GLOBAL_CONTEXT_NAME:
            raise ValueError('Impossible de supprimer _global')
        ctx = self.get(name)
        if ctx:
            shutil.rmtree(ctx.path)

    # ─── Hiérarchie ───────────────────────────────────────────────────

    def get_chain(self, name: str) -> list[ContextInfo]:
        """
        Retourner la chaîne d'héritage du context jusqu'à _global.
        Ex : mobile.flutter → [mobile.flutter, mobile, _global]
        """
        chain: list[ContextInfo] = []
        current = self.get(name)
        visited: set[str] = set()
        while current and current.name not in visited:
            chain.append(current)
            visited.add(current.name)
            if not current.parent:
                break
            current = self.get(current.parent)
        return chain

    def get_descendants(self, name: str) -> list[ContextInfo]:
        """Retourner tous les contexts qui ont `name` comme ancêtre"""
        result: list[ContextInfo] = []
        for ctx in self.list_active():
            chain_names = [c.name for c in self.get_chain(ctx.name)]
            if name in chain_names and ctx.name != name:
                result.append(ctx)
        return result

    # ─── Auto-détection ───────────────────────────────────────────────

    def auto_detect(self, project_root: Path) -> list[str]:
        """
        Détecter le(s) context(s) pertinent(s) pour un projet.
        Retourne la liste des contexts à activer (ordonnés du plus général au plus spécifique).
        """
        candidates: list[tuple[str, int]] = []  # (name, score)

        # Tester contre chaque template + context actif
        all_definitions = list(self._all_context_configs())
        for name, config in all_definitions:
            score = self._score_detection(config, project_root)
            if score > 0:
                candidates.append((name, score))

        if not candidates:
            return [GLOBAL_CONTEXT_NAME]

        # Garder le meilleur match
        candidates.sort(key=lambda x: x[1], reverse=True)
        best_name, _ = candidates[0]

        # Construire la chaîne d'activation
        # Note : on doit travailler à partir du config (template ou actif)
        chain = self._build_activation_chain(best_name, dict(all_definitions))
        return chain

    def _build_activation_chain(self, name: str, all_configs: dict) -> list[str]:
        """Construire la chaîne d'activation [_global, parent, ..., name]"""
        chain: list[str] = []
        current = name
        visited: set[str] = set()
        while current and current not in visited:
            chain.append(current)
            visited.add(current)
            cfg = all_configs.get(current)
            current = cfg.get('parent') if cfg else None
        return list(reversed(chain))

    def _all_context_configs(self):
        """Itère sur tous les configs (templates + actifs)"""
        for ctx_dir in sorted(self.contexts_home.iterdir()) if self.contexts_home.exists() else []:
            if ctx_dir.is_dir():
                cfg = self._read_config(ctx_dir)
                if cfg:
                    yield cfg['name'], cfg
        if self.templates_dir.exists():
            for tpl_dir in sorted(self.templates_dir.iterdir()):
                if tpl_dir.is_dir():
                    cfg = self._read_config(tpl_dir)
                    if cfg:
                        yield cfg['name'], cfg

    @staticmethod
    def _score_detection(config: dict, project_root: Path) -> int:
        """Scorer combien le projet matche les heuristiques d'un context"""
        detection = config.get('detection', {})
        if not detection:
            return 0
        score = 0
        for required_file in detection.get('files', []):
            if (project_root / required_file).exists():
                score += 10
        for keyword in detection.get('required_in_pubspec', []):
            pub = project_root / 'pubspec.yaml'
            if pub.exists() and keyword in pub.read_text(errors='ignore'):
                score += 5
        return score

    # ─── Export / Import ──────────────────────────────────────────────

    def export(self, name: str, target_archive: Path):
        """Exporter un context en archive .tar.gz pour partage"""
        ctx = self.get(name)
        if not ctx:
            raise ValueError(f'Context inexistant : {name}')
        import tarfile
        with tarfile.open(target_archive, 'w:gz') as tar:
            tar.add(ctx.path, arcname=name)

    def install(self, archive_path: Path) -> ContextInfo:
        """Installer un context depuis une archive partagée"""
        import tarfile
        with tarfile.open(archive_path, 'r:gz') as tar:
            members = tar.getmembers()
            top_level = {m.name.split('/')[0] for m in members}
            if len(top_level) != 1:
                raise ValueError('Archive invalide : un seul context attendu')
            name = next(iter(top_level))
            if self.get(name):
                raise ValueError(f'Context "{name}" existe déjà — utiliser fork')
            tar.extractall(self.contexts_home)
        return self.get(name)

    # ─── Helpers ──────────────────────────────────────────────────────

    @staticmethod
    def _read_config(ctx_dir: Path) -> dict | None:
        config_file = ctx_dir / 'config.yaml'
        if not config_file.exists():
            return None
        try:
            return yaml.safe_load(config_file.read_text(encoding='utf-8'))
        except (yaml.YAMLError, OSError):
            return None

    @staticmethod
    def _write_config(ctx_dir: Path, config: dict):
        config_file = ctx_dir / 'config.yaml'
        config_file.write_text(
            yaml.dump(config, allow_unicode=True, default_flow_style=False, sort_keys=False),
            encoding='utf-8',
        )

    @staticmethod
    def _count_decisions(ctx_dir: Path) -> int:
        log = ctx_dir / 'decisions.log'
        if not log.exists():
            return 0
        return sum(1 for line in log.read_text(errors='ignore').splitlines() if line.startswith('['))
```

## Templates Bundled — Format

Chaque template (`src/workflow/templates/contexts/<name>/`) contient :
- `config.yaml` (avec `detection`, `parent`, `tags`, `models`)
- `tech-stack-defaults.json`
- `design-defaults.json` (pour ceux avec UI)
- `allowed-commands-defaults.json`
- **PAS de `skills/` ni `decisions.log`** — ils s'enrichissent avec l'usage

## Intégration avec les Autres Modules

### `SkillManager` (tâche 1.5) — context-aware

```python
class SkillManager:
    def __init__(self, project_root: str, registry: ContextRegistry):
        self.registry = registry
        ...

    def search(self, query: str, active_contexts: list[str], max_results: int = 5):
        """Cherche dans les skills des contexts actifs (du plus spécifique au plus général)"""
        all_skill_files: list[Path] = []
        # Du plus spécifique au plus général
        for ctx_name in active_contexts:
            ctx = self.registry.get(ctx_name)
            if ctx:
                all_skill_files.extend((ctx.path / 'skills').glob('*.md'))
        # + skills strictement projet
        all_skill_files.extend((self.project_root / '.workflow' / 'skills').glob('*.md'))
        ...
```

### `DecisionsLog` (tâche 1.6) — context-scoped

Décisions stockées dans le decisions.log du context approprié. Une décision projet reste dans `.workflow/decisions.log` ; une décision globale dev va dans `~/.workflow/contexts/_global/decisions.log`.

### `LLMContextLoader` (Phase 2 — était `ContextManager`)

Charge skills + decisions de **toute la chaîne** des contexts actifs, avec scoring pondéré par spécificité.

### `MCPServer` (Phase 6) — outils contexts

```
workflow_context_list()
workflow_context_list_available()
workflow_context_create_from_template(name)
workflow_context_create_custom(name, parent, description)
workflow_context_activate(name, scope='project')
workflow_context_export(name, path)
workflow_context_install(archive_path)
workflow_context_fork(source, target)
workflow_context_delete(name)
workflow_context_promote_skill(skill_id, from_context, to_context)
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | Au démarrage, `_ensure_global()` crée `~/.workflow/contexts/_global/` s'il n'existe pas | ⬜ |
| 2 | `list_active()` retourne uniquement les contexts instanciés (pas templates) | ⬜ |
| 3 | `list_available_templates()` retourne les templates bundled non encore instanciés | ⬜ |
| 4 | `create_from_template('mobile.flutter')` crée mobile + mobile.flutter (parent récursif) | ⬜ |
| 5 | `create_custom(name, parent)` refuse si parent inexistant | ⬜ |
| 6 | `fork(source, target)` clone skills + decisions, met à jour `forked_from` | ⬜ |
| 7 | `delete('_global')` lève une erreur | ⬜ |
| 8 | `get_chain('mobile.flutter')` retourne `[mobile.flutter, mobile, _global]` | ⬜ |
| 9 | `auto_detect()` sur un dossier avec pubspec.yaml + flutter: → propose mobile.flutter | ⬜ |
| 10 | `auto_detect()` sur dossier vide → retourne `[_global]` | ⬜ |
| 11 | `export()` produit une archive .tar.gz contenant tout le context | ⬜ |
| 12 | `install()` refuse si un context du même nom existe déjà | ⬜ |
| 13 | Le schéma `context.schema.json` est valide JSON Schema Draft 2020-12 | ⬜ |
| 14 | Tests : 12 cas couvrent create, fork, delete, chain, detect, export/import | ⬜ |

## Notes d'Implémentation

- **Sécurité de l'install** : valider le contenu d'une archive avant extraction (pas de `..` dans les paths, pas de symlinks).
- **Migration** : si Workflow update les templates bundled, l'utilisateur peut faire `workflow context update mobile.flutter` pour merger les nouveautés (interactif).
- **Pas de cycle** : `parent` doit former un DAG. `_ensure_no_cycle()` à valider lors de la création custom.
- **Performance** : `list_active()` et `auto_detect()` peuvent être cachés en mémoire si appelés souvent dans la même session.
- **Marketplace** : `workflow-hub` (Phase 8) consommera `export()` + `install()` directement.
