# Phase 2 — Tâche 2.4 : CodePatcher.py

## Objectif

Créer `CodePatcher.py` — la couche d'application chirurgicale de modifications de code. **Workflow ne régénère JAMAIS un fichier entier** pour appliquer une modification. Edits via search/replace blocks (par défaut), ou AST patches (tree-sitter, fallback ambigu).

> **Pilier load-bearing #5.** Régénérer des fichiers entiers est techniquement médiocre — coût tokens × 10, hallucinations sur portions inchangées, latence absurde. Tous les agents Hermes-tier (Aider, Cursor, Claude Code) le font depuis 2024. Différer ce module en v1.5 condamne le MVP.

## Dépendances

- Phase 1 ✅
- Tâche 2.1 ✅ (`LLMProvider`)

## Fichiers à Créer

- `src/workflow/tools/code_patcher.py` [CRÉER]
- `tests/unit/test_code_patcher.py` [CRÉER]
- `pyproject.toml` : ajouter `tree-sitter`, `tree-sitter-languages`

## Stratégies (par ordre de priorité)

### 1. Création de fichier — `### CRÉER chemin`

Format simple — bloc complet qui remplace tout :

```
### CRÉER src/auth/login.py
```python
def login(email: str, password: str) -> Token:
    ...
```
```

### 2. Modification — `<<<<<<< SEARCH ... >>>>>>> REPLACE`

Format aider-style, prouvé :

```
### MODIFIER src/auth/login.py
<<<<<<< SEARCH
def login(email: str, password: str) -> Token:
    user = db.users.find_one({"email": email})
    if not user:
        return None
=======
def login(email: str, password: str) -> Token:
    user = db.users.find_one({"email": email})
    if not user:
        raise UserNotFoundError(email)
>>>>>>> REPLACE
```

Le bloc SEARCH doit être présent **exactement** dans le fichier. Si plusieurs occurrences, on prend la première — donc le bloc doit inclure assez de contexte pour être unique (≥3 lignes recommandé).

### 3. Fallback AST (tree-sitter)

Si le SEARCH/REPLACE échoue (texte introuvable, ambigu, ou whitespace mismatch), tentative via tree-sitter :

```
Le LLM peut aussi annoter :
### MODIFIER src/auth/login.py
@@AST_REPLACE function:login@@
```python
def login(email: str, password: str) -> Token:
    user = db.users.find_one({"email": email})
    if not user:
        raise UserNotFoundError(email)
    ...
```
```

`CodePatcher` localise la fonction `login` via tree-sitter, remplace son corps. Marche même si le whitespace a changé.

## Implémentation

```python
# src/workflow/tools/code_patcher.py
import re
from pathlib import Path
from dataclasses import dataclass
from typing import Literal

try:
    import tree_sitter_languages
    TREE_SITTER_AVAILABLE = True
except ImportError:
    TREE_SITTER_AVAILABLE = False


PatchAction = Literal['CREATE', 'MODIFY', 'AST_REPLACE']


@dataclass
class Patch:
    action: PatchAction
    file_path: str
    content: str | None = None        # Pour CREATE + AST_REPLACE
    search: str | None = None         # Pour MODIFY
    replace: str | None = None        # Pour MODIFY
    ast_target: str | None = None     # Pour AST_REPLACE (ex: "function:login")


@dataclass
class PatchResult:
    success: bool
    file_path: str
    action: PatchAction
    error: str | None = None
    bytes_changed: int = 0


class CodePatcher:

    # Patterns de parsing
    CREATE_PATTERN = re.compile(
        r'^### CRÉER (?P<path>.+?)\n```(?:\w+)?\n(?P<content>.*?)```',
        re.MULTILINE | re.DOTALL,
    )
    SEARCH_REPLACE_PATTERN = re.compile(
        r'^### MODIFIER (?P<path>.+?)\n'
        r'(?:(?!^### )(?:.|\n))*?'
        r'<<<<<<< SEARCH\n(?P<search>.*?)\n=======\n(?P<replace>.*?)\n>>>>>>> REPLACE',
        re.MULTILINE | re.DOTALL,
    )
    AST_REPLACE_PATTERN = re.compile(
        r'^### MODIFIER (?P<path>.+?)\n@@AST_REPLACE (?P<target>[^@]+)@@\n'
        r'```(?:\w+)?\n(?P<content>.*?)```',
        re.MULTILINE | re.DOTALL,
    )

    def __init__(self, project_root: str):
        self.project_root = Path(project_root).resolve()

    # ─── Parsing ──────────────────────────────────────────────────────

    def parse_patches(self, llm_response: str) -> list[Patch]:
        """Extraire tous les patches de la réponse LLM"""
        patches: list[Patch] = []

        for m in self.CREATE_PATTERN.finditer(llm_response):
            patches.append(Patch(
                action='CREATE',
                file_path=m.group('path').strip(),
                content=m.group('content'),
            ))

        for m in self.AST_REPLACE_PATTERN.finditer(llm_response):
            patches.append(Patch(
                action='AST_REPLACE',
                file_path=m.group('path').strip(),
                ast_target=m.group('target').strip(),
                content=m.group('content'),
            ))

        for m in self.SEARCH_REPLACE_PATTERN.finditer(llm_response):
            patches.append(Patch(
                action='MODIFY',
                file_path=m.group('path').strip(),
                search=m.group('search'),
                replace=m.group('replace'),
            ))

        return patches

    # ─── Application ──────────────────────────────────────────────────

    async def apply(self, patches: list[Patch]) -> list[PatchResult]:
        """Appliquer une liste de patches dans l'ordre, retourner les résultats"""
        results = []
        for p in patches:
            try:
                if p.action == 'CREATE':
                    result = await self._apply_create(p)
                elif p.action == 'MODIFY':
                    result = await self._apply_modify(p)
                elif p.action == 'AST_REPLACE':
                    result = await self._apply_ast_replace(p)
                else:
                    result = PatchResult(
                        success=False,
                        file_path=p.file_path,
                        action=p.action,
                        error=f'Action inconnue : {p.action}',
                    )
                results.append(result)
            except Exception as e:
                results.append(PatchResult(
                    success=False,
                    file_path=p.file_path,
                    action=p.action,
                    error=str(e),
                ))
        return results

    async def _apply_create(self, p: Patch) -> PatchResult:
        full_path = self.project_root / p.file_path
        if full_path.exists():
            return PatchResult(
                success=False,
                file_path=p.file_path,
                action='CREATE',
                error='Le fichier existe déjà — utiliser MODIFIER',
            )
        full_path.parent.mkdir(parents=True, exist_ok=True)
        full_path.write_text(p.content or '', encoding='utf-8')
        return PatchResult(
            success=True,
            file_path=p.file_path,
            action='CREATE',
            bytes_changed=len(p.content or ''),
        )

    async def _apply_modify(self, p: Patch) -> PatchResult:
        full_path = self.project_root / p.file_path
        if not full_path.exists():
            return PatchResult(
                success=False,
                file_path=p.file_path,
                action='MODIFY',
                error='Fichier introuvable',
            )

        original = full_path.read_text(encoding='utf-8')

        # Recherche stricte
        if p.search not in original:
            # Fallback : recherche tolérante au whitespace
            relaxed_search = self._normalize_whitespace(p.search or '')
            relaxed_original = self._normalize_whitespace(original)
            if relaxed_search not in relaxed_original:
                return PatchResult(
                    success=False,
                    file_path=p.file_path,
                    action='MODIFY',
                    error=(
                        f'Bloc SEARCH introuvable dans le fichier. '
                        f'Considérer @@AST_REPLACE@@ pour ce changement.'
                    ),
                )
            # Found via relaxed match — use indexes from original
            start = self._find_relaxed(original, p.search or '')
            modified = original[:start] + (p.replace or '') + original[start + len(p.search or ''):]
        else:
            modified = original.replace(p.search or '', p.replace or '', 1)

        # Détecter ambiguïté : plusieurs occurrences
        if (p.search or '') in original and original.count(p.search or '') > 1:
            return PatchResult(
                success=False,
                file_path=p.file_path,
                action='MODIFY',
                error=(
                    f'Bloc SEARCH ambigu : trouvé {original.count(p.search or "")} fois. '
                    f'Étendre le contexte (≥3 lignes uniques).'
                ),
            )

        full_path.write_text(modified, encoding='utf-8')
        return PatchResult(
            success=True,
            file_path=p.file_path,
            action='MODIFY',
            bytes_changed=len(p.replace or '') - len(p.search or ''),
        )

    async def _apply_ast_replace(self, p: Patch) -> PatchResult:
        if not TREE_SITTER_AVAILABLE:
            return PatchResult(
                success=False,
                file_path=p.file_path,
                action='AST_REPLACE',
                error='tree-sitter non disponible — installer tree-sitter-languages',
            )

        full_path = self.project_root / p.file_path
        if not full_path.exists():
            return PatchResult(
                success=False,
                file_path=p.file_path,
                action='AST_REPLACE',
                error='Fichier introuvable',
            )

        # Format de target : "function:login" | "class:User" | "method:User.save"
        target_kind, target_name = (p.ast_target or '').split(':', 1)
        language = self._detect_language(p.file_path)
        if not language:
            return PatchResult(
                success=False,
                file_path=p.file_path,
                action='AST_REPLACE',
                error=f'Langage non supporté pour : {p.file_path}',
            )

        parser = tree_sitter_languages.get_parser(language)
        original = full_path.read_text(encoding='utf-8')
        tree = parser.parse(original.encode('utf-8'))

        node = self._find_ast_node(tree.root_node, target_kind, target_name, language)
        if not node:
            return PatchResult(
                success=False,
                file_path=p.file_path,
                action='AST_REPLACE',
                error=f'Cible AST introuvable : {p.ast_target}',
            )

        modified = (
            original[:node.start_byte] + (p.content or '') + original[node.end_byte:]
        )
        full_path.write_text(modified, encoding='utf-8')
        return PatchResult(
            success=True,
            file_path=p.file_path,
            action='AST_REPLACE',
            bytes_changed=len(p.content or '') - (node.end_byte - node.start_byte),
        )

    # ─── Helpers ──────────────────────────────────────────────────────

    @staticmethod
    def _normalize_whitespace(text: str) -> str:
        return re.sub(r'\s+', ' ', text).strip()

    @staticmethod
    def _find_relaxed(haystack: str, needle: str) -> int:
        """Recherche tolérante : compare sans whitespace exacte mais retourne l'index original"""
        # Implémentation simplifiée — robuste : utilise diff algo
        normalized_needle = CodePatcher._normalize_whitespace(needle)
        for i in range(len(haystack)):
            chunk = haystack[i:i + len(needle) + 50]
            if normalized_needle in CodePatcher._normalize_whitespace(chunk):
                return i
        return -1

    @staticmethod
    def _detect_language(file_path: str) -> str | None:
        ext_map = {
            '.py': 'python',
            '.js': 'javascript',
            '.ts': 'typescript',
            '.tsx': 'tsx',
            '.jsx': 'javascript',
            '.go': 'go',
            '.rs': 'rust',
            '.java': 'java',
            '.rb': 'ruby',
        }
        return ext_map.get(Path(file_path).suffix)

    @staticmethod
    def _find_ast_node(root, kind: str, name: str, language: str):
        """Trouver un nœud AST par kind (function/class/method) et nom"""
        kind_map = {
            'python': {
                'function': 'function_definition',
                'class': 'class_definition',
                'method': 'function_definition',  # within class
            },
            'javascript': {
                'function': 'function_declaration',
                'class': 'class_declaration',
                'method': 'method_definition',
            },
        }
        node_type = kind_map.get(language, {}).get(kind)
        if not node_type:
            return None

        def walk(node):
            if node.type == node_type:
                # Trouver le nom du nœud
                name_node = next(
                    (c for c in node.children if c.type == 'identifier'),
                    None,
                )
                if name_node and name_node.text.decode('utf-8') == name:
                    return node
            for child in node.children:
                result = walk(child)
                if result:
                    return result
            return None

        return walk(root)
```

## Intégration avec `ExecutionLoop` (Phase 3)

```python
async def run(self, task_ctx: dict) -> dict:
    ...
    raw_response = await self.llm.ask(prompt, role='code_generation')

    # Parser et appliquer les patches
    patches = self.code_patcher.parse_patches(raw_response)
    if not patches:
        self.io.print_warning('Aucun patch détecté dans la réponse LLM.')
        return {'success': False, 'error': 'no_patches'}

    results = await self.code_patcher.apply(patches)
    failed = [r for r in results if not r.success]
    if failed:
        # Retry avec les erreurs en feedback
        last_error = {'output': '\n'.join(f'{r.file_path}: {r.error}' for r in failed)}
        ...
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `parse_patches()` extrait correctement un `### CRÉER` avec son contenu | ⬜ |
| 2 | `parse_patches()` extrait un `<<<<<<< SEARCH ... >>>>>>> REPLACE` | ⬜ |
| 3 | `parse_patches()` extrait un `@@AST_REPLACE function:foo@@` | ⬜ |
| 4 | `apply()` CREATE refuse si le fichier existe déjà | ⬜ |
| 5 | `apply()` MODIFY échoue avec message clair si SEARCH introuvable | ⬜ |
| 6 | `apply()` MODIFY échoue avec message clair si SEARCH ambigu (>1 occurrence) | ⬜ |
| 7 | `apply()` MODIFY tolère un mismatch de whitespace mineur (fallback relaxed) | ⬜ |
| 8 | `apply()` AST_REPLACE remplace une fonction Python correctement (test avec mock tree-sitter) | ⬜ |
| 9 | `apply()` ne touche jamais à un fichier hors `project_root` | ⬜ |
| 10 | Les patches sont appliqués dans l'ordre — un échec n'interrompt pas les suivants | ⬜ |
| 11 | `PatchResult.bytes_changed` est positif/négatif/zéro selon le delta | ⬜ |
| 12 | Tests : 8 cas (CREATE OK, CREATE conflit, MODIFY OK, MODIFY 404, MODIFY ambigu, MODIFY whitespace, AST OK, AST 404) | ⬜ |

## Notes d'Implémentation

- **Sécurité** : valider que `project_root / file_path` reste sous `project_root` (pas de `../` malicieux).
- **Atomicité** : l'écriture via `Path.write_text()` n'est pas atomique. Pour Phase 9 (robustesse), passer par `FileSystem.atomic_write()` (write-temp + rename).
- **Performance** : éviter de relire le fichier plusieurs fois si plusieurs patches sur le même fichier — batcher.
- **Feedback** : `PatchResult.error` doit être actionnable par le LLM en retry (ex : "ambigu, étends le contexte" plutôt que "match failed").
