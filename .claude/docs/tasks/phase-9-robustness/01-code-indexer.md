# Phase 9 — Tâche 9.1 : CodeIndexer.py

## Objectif

Créer `CodeIndexer.py` — un index sémantique des fonctions, classes, exports et imports du projet, mis à jour en continu. Permet à `ContextManager` de répondre rapidement à "où est définie X ?" sans relire tout le projet, et à `ExecutionLoop` de détecter les conflits avant l'écriture.

> **Pilier de robustesse**. Sur un projet de 200+ fichiers, charger tout en contexte est insoutenable. L'indexer permet de charger uniquement les fichiers et symboles pertinents pour la tâche courante.

## Dépendances

- Phase 1 ✅ (`FileSystem`)
- Tâche 2.4 ✅ (tree-sitter déjà installé pour `CodePatcher`)

## Fichiers à Créer

- `src/workflow/tools/code_indexer.py` [CRÉER]
- `tests/unit/test_code_indexer.py` [CRÉER]

## Modèle de Données — `code-index.json`

```json
{
  "schema_version": "1.0.0",
  "last_full_scan": "2026-04-15T14:30:00Z",
  "files": {
    "src/auth/login.py": {
      "language": "python",
      "last_modified": "2026-04-15T14:25:00Z",
      "hash": "sha256:abc123...",
      "symbols": [
        { "name": "login", "type": "function", "line": 12, "exported": true },
        { "name": "validate_password", "type": "function", "line": 45, "exported": false },
        { "name": "AuthError", "type": "class", "line": 8, "exported": true }
      ],
      "imports": [
        { "module": "bcrypt", "external": true },
        { "module": "src.db", "external": false, "symbols": ["users_collection"] }
      ]
    }
  },
  "by_symbol": {
    "login": ["src/auth/login.py:12"],
    "AuthError": ["src/auth/login.py:8"]
  }
}
```

## Stratégie de Mise à Jour

1. **Plein scan initial** — au premier `workflow init` ou `workflow index --rebuild`
2. **Mise à jour incrémentale** — déclenché par :
   - `WatchMode` quand un fichier change
   - `CodePatcher` après une application réussie
   - `SyncChecker.check()` au boot, sur les fichiers `git diff` depuis le dernier scan
3. **Recherche** — `query()` retourne les fichiers/symboles pertinents pour une requête (matching nom + variantes LLM via `role='fast'`)

## Implémentation

```python
# src/workflow/tools/code_indexer.py
import asyncio
import hashlib
import json
import re
import subprocess
from pathlib import Path
from datetime import datetime, timezone

import tree_sitter_languages

from workflow.tools.filesystem import FileSystem
from workflow.llm.llm_provider import LLMProvider

INDEX_FILE = 'code-index.json'

LANGUAGE_BY_EXT = {
    '.py': 'python',
    '.js': 'javascript',
    '.ts': 'typescript',
    '.tsx': 'tsx',
    '.go': 'go',
    '.rs': 'rust',
    '.java': 'java',
    '.rb': 'ruby',
}

IGNORED_DIRS = {'node_modules', '.git', '.venv', '__pycache__', 'dist', 'build', '.workflow'}


class CodeIndexer:
    def __init__(self, project_root: str, llm: LLMProvider | None = None):
        self.project_root = Path(project_root).resolve()
        self.fs = FileSystem(project_root)
        self.llm = llm
        self._index: dict | None = None

    async def _load(self) -> dict:
        if self._index is not None:
            return self._index
        data = await self.fs.read_json(INDEX_FILE)
        self._index = data or {
            'schema_version': '1.0.0',
            'last_full_scan': None,
            'files': {},
            'by_symbol': {},
        }
        return self._index

    async def _save(self):
        await self.fs.write_json_atomic(INDEX_FILE, self._index)

    # ─── Scan complet ─────────────────────────────────────────────────

    async def rebuild(self) -> dict:
        """Reconstruire l'index complet"""
        self._index = {
            'schema_version': '1.0.0',
            'last_full_scan': datetime.now(timezone.utc).isoformat(),
            'files': {},
            'by_symbol': {},
        }

        for file_path in self._iter_source_files():
            await self._index_file(file_path)

        self._rebuild_symbol_map()
        await self._save()
        return {
            'files_indexed': len(self._index['files']),
            'symbols': len(self._index['by_symbol']),
        }

    def _iter_source_files(self):
        for root, dirs, files in self._os_walk():
            dirs[:] = [d for d in dirs if d not in IGNORED_DIRS]
            for f in files:
                ext = Path(f).suffix
                if ext in LANGUAGE_BY_EXT:
                    yield Path(root) / f

    def _os_walk(self):
        import os
        for root, dirs, files in os.walk(self.project_root):
            yield root, dirs, files

    # ─── Indexation d'un fichier ─────────────────────────────────────

    async def _index_file(self, file_path: Path):
        rel_path = str(file_path.relative_to(self.project_root))
        language = LANGUAGE_BY_EXT.get(file_path.suffix)
        if not language:
            return

        try:
            content = file_path.read_text(encoding='utf-8')
        except (OSError, UnicodeDecodeError):
            return

        file_hash = hashlib.sha256(content.encode()).hexdigest()

        # Skip si non modifié depuis dernier scan
        existing = self._index['files'].get(rel_path)
        if existing and existing.get('hash') == 'sha256:' + file_hash:
            return

        symbols = self._extract_symbols(content, language)
        imports = self._extract_imports(content, language)

        self._index['files'][rel_path] = {
            'language': language,
            'last_modified': datetime.fromtimestamp(
                file_path.stat().st_mtime, tz=timezone.utc
            ).isoformat(),
            'hash': 'sha256:' + file_hash,
            'symbols': symbols,
            'imports': imports,
        }

    def _extract_symbols(self, content: str, language: str) -> list[dict]:
        """Extraire les symboles via tree-sitter"""
        try:
            parser = tree_sitter_languages.get_parser(language)
            tree = parser.parse(content.encode('utf-8'))
        except Exception:
            return []

        symbols = []
        kind_map = {
            'python': {
                'function_definition': 'function',
                'class_definition': 'class',
            },
            'javascript': {
                'function_declaration': 'function',
                'class_declaration': 'class',
                'method_definition': 'method',
            },
            'typescript': {
                'function_declaration': 'function',
                'class_declaration': 'class',
                'interface_declaration': 'interface',
                'type_alias_declaration': 'type',
            },
        }
        node_kinds = kind_map.get(language, {})

        def walk(node):
            if node.type in node_kinds:
                name_node = next(
                    (c for c in node.children if c.type == 'identifier' or c.type == 'type_identifier'),
                    None,
                )
                if name_node:
                    name = name_node.text.decode('utf-8')
                    symbols.append({
                        'name': name,
                        'type': node_kinds[node.type],
                        'line': name_node.start_point[0] + 1,
                        'exported': not name.startswith('_'),
                    })
            for child in node.children:
                walk(child)

        walk(tree.root_node)
        return symbols

    def _extract_imports(self, content: str, language: str) -> list[dict]:
        """Extraire les imports — implémentation simple par regex (par langage)"""
        imports = []
        if language == 'python':
            patterns = [
                r'^from\s+([\w.]+)\s+import\s+(.+)$',
                r'^import\s+([\w.]+)$',
            ]
            for pat in patterns:
                for m in re.finditer(pat, content, re.MULTILINE):
                    module = m.group(1)
                    imports.append({
                        'module': module,
                        'external': not (module.startswith('.') or module.startswith('src.')),
                    })
        elif language in ('javascript', 'typescript', 'tsx'):
            for m in re.finditer(
                r'import\s+.*?\s+from\s+[\'"]([^\'\"]+)[\'"]', content
            ):
                module = m.group(1)
                imports.append({
                    'module': module,
                    'external': not (module.startswith('.') or module.startswith('/')),
                })
        return imports

    def _rebuild_symbol_map(self):
        """Reconstruire l'index by_symbol depuis les files"""
        by_symbol: dict[str, list[str]] = {}
        for file_path, file_data in self._index['files'].items():
            for sym in file_data.get('symbols', []):
                key = sym['name']
                by_symbol.setdefault(key, []).append(f'{file_path}:{sym["line"]}')
        self._index['by_symbol'] = by_symbol

    # ─── Mise à jour incrémentale ─────────────────────────────────────

    async def update_file(self, rel_path: str):
        """Mettre à jour l'index pour un fichier (appelé par WatchMode et CodePatcher)"""
        await self._load()
        full_path = self.project_root / rel_path
        if not full_path.exists():
            self._index['files'].pop(rel_path, None)
        else:
            await self._index_file(full_path)
        self._rebuild_symbol_map()
        await self._save()

    async def update_since(self, since_iso: str):
        """Mettre à jour les fichiers modifiés depuis une date (via git diff)"""
        try:
            result = subprocess.run(
                ['git', 'diff', '--name-only', f'@{{{since_iso}}}', 'HEAD'],
                cwd=self.project_root,
                capture_output=True,
                text=True,
                timeout=10,
            )
            files = result.stdout.strip().split('\n') if result.stdout else []
            for f in files:
                if f and Path(f).suffix in LANGUAGE_BY_EXT:
                    await self.update_file(f)
        except (subprocess.SubprocessError, OSError):
            pass

    # ─── Recherche ────────────────────────────────────────────────────

    async def query(self, query_text: str, max_results: int = 10) -> list[dict]:
        """
        Rechercher symboles + fichiers pertinents.
        1. Match exact sur by_symbol
        2. Match préfixe / camelCase / snake_case
        3. Si LLM disponible, expansion sémantique via role='fast'
        """
        await self._load()
        results: list[dict] = []
        seen: set[str] = set()
        q = query_text.lower()

        # 1. Match exact
        for sym, locations in self._index['by_symbol'].items():
            if sym.lower() == q and sym not in seen:
                for loc in locations:
                    results.append({'symbol': sym, 'location': loc, 'match': 'exact'})
                seen.add(sym)

        # 2. Match préfixe / contient
        if len(results) < max_results:
            for sym, locations in self._index['by_symbol'].items():
                if sym in seen:
                    continue
                if q in sym.lower() or sym.lower() in q:
                    for loc in locations:
                        results.append({'symbol': sym, 'location': loc, 'match': 'partial'})
                    seen.add(sym)
                    if len(results) >= max_results:
                        break

        # 3. Expansion sémantique LLM (si dispo et résultats pauvres)
        if self.llm and len(results) < 3:
            variants = await self._llm_query_variants(query_text)
            for v in variants:
                v_lower = v.lower()
                for sym, locations in self._index['by_symbol'].items():
                    if sym in seen:
                        continue
                    if v_lower in sym.lower() or sym.lower() in v_lower:
                        for loc in locations:
                            results.append({
                                'symbol': sym,
                                'location': loc,
                                'match': f'llm_variant:{v}',
                            })
                        seen.add(sym)

        return results[:max_results]

    async def _llm_query_variants(self, query: str) -> list[str]:
        """Demander au LLM 3 variantes synonymes/related du query"""
        prompt = (
            f'Liste 3 noms de variables/fonctions qu\'un développeur '
            f'pourrait utiliser pour le concept "{query}". '
            f'Retourne JSON : ["nom1", "nom2", "nom3"]'
        )
        try:
            response = await self.llm.ask(prompt, role='fast', max_tokens=200)
            return json.loads(response.strip())
        except (json.JSONDecodeError, Exception):
            return []

    # ─── Recherche dans le contenu (ripgrep) ─────────────────────────

    async def grep(self, pattern: str, max_results: int = 50) -> list[dict]:
        """Recherche full-text via ripgrep — utile pour ContextManager.loadOnDemand"""
        try:
            result = subprocess.run(
                ['rg', '--json', '--max-count=2', pattern, str(self.project_root)],
                capture_output=True,
                text=True,
                timeout=10,
            )
        except (FileNotFoundError, subprocess.SubprocessError):
            return []

        matches = []
        for line in result.stdout.split('\n'):
            if not line.strip():
                continue
            try:
                obj = json.loads(line)
                if obj.get('type') == 'match':
                    data = obj['data']
                    matches.append({
                        'path': data['path']['text'],
                        'line': data['line_number'],
                        'text': data['lines']['text'].strip(),
                    })
                    if len(matches) >= max_results:
                        break
            except (json.JSONDecodeError, KeyError):
                continue
        return matches
```

## Intégration

**`ContextManager.loadOnDemand()` (Phase 2)** :
```python
async def load_on_demand(self, query: str) -> list[dict]:
    return await self.indexer.query(query, max_results=10)
```

**`WatchMode.on_file_change()` (Phase 7)** :
```python
async def on_file_change(self, path: str):
    await self.indexer.update_file(path)
```

**`CodePatcher.apply()` (Phase 2.4)** — après chaque patch appliqué :
```python
await self.indexer.update_file(p.file_path)
```

**`SyncChecker.check()` (Phase 1.8)** — au boot :
```python
last_scan = self.index['last_full_scan']
if last_scan:
    await self.indexer.update_since(last_scan)
else:
    await self.indexer.rebuild()
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `rebuild()` parcourt récursivement le projet en ignorant `node_modules`, `.git`, etc. | ⬜ |
| 2 | `_extract_symbols()` détecte fonctions, classes, méthodes via tree-sitter | ⬜ |
| 3 | `_extract_imports()` détecte les imports Python et JS/TS | ⬜ |
| 4 | `update_file()` met à jour un seul fichier sans relire tout | ⬜ |
| 5 | `update_file()` supprime l'entrée si le fichier n'existe plus | ⬜ |
| 6 | `query('login')` retourne les locations exactes du symbole | ⬜ |
| 7 | `query('aut')` retourne `authenticate`, `auth`, etc. (match partiel) | ⬜ |
| 8 | `grep('pattern')` utilise ripgrep et retourne les locations | ⬜ |
| 9 | Le hash sha256 évite la ré-indexation si le fichier n'a pas changé | ⬜ |
| 10 | `update_since(iso)` utilise git diff pour identifier les fichiers modifiés | ⬜ |
| 11 | `_llm_query_variants()` est utilisé seulement si <3 résultats locaux | ⬜ |
| 12 | Tests : 6 langages (Python, JS, TS, Go, Rust, Java) avec mocks tree-sitter | ⬜ |

## Notes d'Implémentation

- **Performance** : `rebuild()` sur un projet de 1000 fichiers doit prendre < 30s. Si tree-sitter est trop lent, fallback sur regex pour Python/JS (déjà en place pour les imports).
- **Quotas LLM** : `_llm_query_variants` est gated derrière `len(results) < 3` pour éviter les appels superflus. Cache les variants par query si nécessaire (Phase ultérieure).
- **Conflits CodePatcher** : avant d'appliquer un patch sur `function login`, vérifier qu'il n'y a pas un autre `login` dans le projet (warning utilisateur).
- **Cross-projet** : non couvert ici. Pour `WorkflowLibrary` (tâche 9.2), un index global cross-projet sera nécessaire.
