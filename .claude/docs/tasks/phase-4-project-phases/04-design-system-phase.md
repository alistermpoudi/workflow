# Phase 2 — Tâche 2.7 : DesignSystemPhase.py + DesignReviewer.py

## Objectif

Ajouter une phase `DESIGN` entre `DISCOVERY` et `SPECIFICATION`. À partir du style choisi en Discovery et des références de l'utilisateur, le LLM génère un **design system complet** persisté dans `.workflow/`. Toutes les tâches UI du projet référencent ce design system — le développeur sait exactement avec quels tokens coder chaque écran.

## Dépendances

- Tâche 2.4 ✅ (`DiscoveryPhase` — fournit `design.json` avec style + références)

## Fichiers à Créer

- `src/workflow/phases/design_system_phase.py` [CRÉER]
- `src/workflow/tools/design_reviewer.py` [CRÉER]

## Mise à Jour PHASE_ORDER

```python
# src/workflow/core/phase_manager.py — METTRE À JOUR
PHASE_ORDER = [
    'DISCOVERY',
    'DESIGN',          # ← NOUVEAU
    'SPECIFICATION',
    'ARCHITECTURE',
    'VALIDATION',
    'ACTIVE',
]
```

## Artefacts Générés

### `.workflow/design-system.json`

```json
{
  "tokens": {
    "colors": {
      "primary":        "#2563eb",
      "primary-hover":  "#1d4ed8",
      "background":     "#ffffff",
      "surface":        "#f8fafc",
      "surface-raised": "#f1f5f9",
      "text":           "#0f172a",
      "text-muted":     "#64748b",
      "border":         "#e2e8f0",
      "success":        "#16a34a",
      "error":          "#dc2626",
      "warning":        "#d97706",
      "info":           "#0284c7"
    },
    "typography": {
      "font-sans":  "Inter, system-ui, sans-serif",
      "font-mono":  "JetBrains Mono, monospace",
      "sizes": {
        "xs": "12px", "sm": "14px", "base": "16px",
        "lg": "18px", "xl": "24px", "2xl": "32px", "3xl": "48px"
      },
      "weights": { "normal": 400, "medium": 500, "semibold": 600, "bold": 700 },
      "line-heights": { "tight": 1.25, "normal": 1.5, "relaxed": 1.75 }
    },
    "spacing": [0, 4, 8, 12, 16, 20, 24, 32, 40, 48, 64, 80, 96],
    "radius": {
      "none": 0, "sm": 4, "md": 8, "lg": 12, "xl": 16, "2xl": 24, "full": 9999
    },
    "shadows": {
      "sm":  "0 1px 2px rgba(0,0,0,0.05)",
      "md":  "0 4px 6px rgba(0,0,0,0.07)",
      "lg":  "0 10px 15px rgba(0,0,0,0.10)",
      "xl":  "0 20px 25px rgba(0,0,0,0.12)"
    },
    "transitions": { "fast": "150ms ease", "normal": "250ms ease", "slow": "400ms ease" }
  },
  "components": {
    "Button":     { "variants": ["primary", "secondary", "ghost", "danger", "link"], "sizes": ["sm", "md", "lg"] },
    "Input":      { "variants": ["default", "error", "disabled", "readonly"] },
    "Textarea":   { "variants": ["default", "error"] },
    "Select":     { "variants": ["default", "error"] },
    "Card":       { "variants": ["default", "elevated", "bordered", "flat"] },
    "Modal":      { "sizes": ["sm", "md", "lg", "xl", "full"] },
    "Badge":      { "variants": ["success", "warning", "error", "info", "neutral"] },
    "Alert":      { "variants": ["success", "warning", "error", "info"] },
    "Spinner":    { "sizes": ["sm", "md", "lg"] },
    "Sidebar":    { "variants": ["collapsed", "expanded"], "position": ["left", "right"] },
    "Navbar":     { "variants": ["default", "transparent", "bordered"] },
    "Table":      { "variants": ["default", "striped", "bordered"] },
    "Tooltip":    { "positions": ["top", "bottom", "left", "right"] },
    "Dropdown":   { "positions": ["bottom-left", "bottom-right", "top-left", "top-right"] }
  },
  "generatedAt": "2026-05-03T10:00:00Z",
  "style": "minimaliste",
  "references": ["Linear", "Stripe"]
}
```

### `.workflow/screen-flow.md`

```markdown
# Screen Flow — [Nom du projet]

## Écrans

| ID  | Nom              | Route            | Tâche liée | Composants principaux |
|-----|------------------|------------------|------------|-----------------------|
| S01 | Login            | /login           | TASK-003   | Card, Input, Button   |
| S02 | Dashboard        | /                | TASK-004   | Sidebar, Navbar, Card |
| S03 | Détail Client    | /clients/:id     | TASK-007   | Table, Badge, Modal   |

## Navigation

```
S01 (Login) ──── auth OK ───→ S02 (Dashboard)
                               ├── clic client ──→ S03 (Détail Client)
                               └── + Nouveau ────→ Modal (Création)
```

## Règles de Navigation
- Toutes les routes sauf /login et /register nécessitent une session active
- La sidebar est persistante sur toutes les routes authentifiées
```

---

## Implémentation — `DesignSystemPhase`

```python
# src/workflow/phases/design_system_phase.py
import json
from datetime import datetime, timezone

from workflow.core.project_memory import ProjectMemory
from workflow.llm.llm_provider import LLMProvider

DESIGN_SYSTEM_PROMPT = """Tu es un designer UI expert. Génère un design system complet pour cette application.

Style demandé : {style_label} ({style_key})
Références visuelles : {references}
Palette : {color_scheme}
Notes custom : {custom_notes}
Fonctionnalités principales : {features_summary}

Retourne UNIQUEMENT un JSON valide avec cette structure exacte :
{{
  "tokens": {{
    "colors": {{
      "primary": "#...", "primary-hover": "#...",
      "background": "#...", "surface": "#...", "surface-raised": "#...",
      "text": "#...", "text-muted": "#...", "border": "#...",
      "success": "#...", "error": "#...", "warning": "#...", "info": "#..."
    }},
    "typography": {{
      "font-sans": "...", "font-mono": "...",
      "sizes": {{"xs":"12px","sm":"14px","base":"16px","lg":"18px","xl":"24px","2xl":"32px","3xl":"48px"}},
      "weights": {{"normal":400,"medium":500,"semibold":600,"bold":700}},
      "line-heights": {{"tight":1.25,"normal":1.5,"relaxed":1.75}}
    }},
    "spacing": [0,4,8,12,16,20,24,32,40,48,64,80,96],
    "radius": {{"none":0,"sm":4,"md":8,"lg":12,"xl":16,"2xl":24,"full":9999}},
    "shadows": {{"sm":"...","md":"...","lg":"...","xl":"..."}},
    "transitions": {{"fast":"150ms ease","normal":"250ms ease","slow":"400ms ease"}}
  }},
  "components": {{
    "Button": {{"variants": [...], "sizes": [...]}},
    "Input": {{"variants": [...]}},
    "Card": {{"variants": [...]}},
    ...composants pertinents pour ce type d'application
  }}
}}

Adapte les couleurs et tokens au style {style_label} demandé et aux références {references}.
Retourne UNIQUEMENT le JSON."""

SCREEN_FLOW_PROMPT = """Génère le screen flow de cette application en Markdown.

Vision : {vision}
Fonctionnalités v1.0 : {features}

Format attendu :
# Screen Flow

## Écrans
| ID | Nom | Route | Tâche liée | Composants principaux |
(liste tous les écrans de l'application)

## Navigation
(diagramme ASCII des transitions entre écrans)

## Règles de Navigation
(règles d'accès, guards, redirections)"""


class DesignSystemPhase:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.memory = ProjectMemory(project_root)
        self.llm = llm
        self.io = io

    async def run(self) -> dict:
        design = await self.memory.get_design()
        if not design:
            self.io.print_error('Pas de design.json — lance la phase Discovery d\'abord.')
            return {'completed': False}

        self.io.print_header('Phase Design — Génération du Design System')

        existing_ds = await self.memory.get_design_system()
        if existing_ds:
            self.io.print_info('Design system existant détecté.')
            keep = self.io.prompt('Conserver ce design system ? [o/N]', default='o').strip().lower()
            if keep in ('o', 'oui', 'y', 'yes', ''):
                return {'completed': True}

        features = await self.memory.get_features() or {}
        vision = await self.memory.get_vision() or ''
        features_summary = ', '.join(
            f.get('name', '') for f in features.get('v1.0', [])[:8]
        )

        # Générer les tokens + composants
        self.io.print_info('Génération des tokens et composants...')
        raw_ds = await self.llm.ask(
            DESIGN_SYSTEM_PROMPT.format(
                style_label=design.get('styleLabel', 'Minimaliste'),
                style_key=design.get('style', 'minimaliste'),
                references=', '.join(design.get('references') or []) or 'aucune',
                color_scheme=design.get('colorScheme', 'clair'),
                custom_notes=design.get('customNotes') or 'aucune',
                features_summary=features_summary or 'application générale',
            ),
            role='reasoning',
        )

        design_system = self._parse_json(raw_ds)
        if not design_system:
            self.io.print_error('Le LLM n\'a pas retourné de JSON valide.')
            return {'completed': False}

        design_system['generatedAt'] = datetime.now(timezone.utc).isoformat()
        design_system['style'] = design.get('style')
        design_system['references'] = design.get('references', [])

        # Générer le screen flow
        self.io.print_info('Génération du screen flow...')
        screen_flow = await self.llm.ask(
            SCREEN_FLOW_PROMPT.format(
                vision=vision[:800],
                features=json.dumps(features.get('v1.0', [])[:10], ensure_ascii=False),
            ),
            role='reasoning',
        )

        # Afficher le résumé
        self._display_summary(design_system)

        while True:
            choice = self.io.prompt('\n[v]alider | [r]egénérer', default='v').strip().lower()
            if choice in ('v', 'valider', ''):
                break
            elif choice in ('r', 'regénérer'):
                self.io.print_info('Régénération...')
                raw_ds = await self.llm.ask(
                    DESIGN_SYSTEM_PROMPT.format(
                        style_label=design.get('styleLabel', 'Minimaliste'),
                        style_key=design.get('style', 'minimaliste'),
                        references=', '.join(design.get('references') or []),
                        color_scheme=design.get('colorScheme', 'clair'),
                        custom_notes=design.get('customNotes') or 'aucune',
                        features_summary=features_summary,
                    ),
                    role='reasoning',
                )
                design_system = self._parse_json(raw_ds) or design_system
                self._display_summary(design_system)

        await self.memory.save_design_system(design_system)
        await self.memory.save_screen_flow(screen_flow)

        self.io.print_success(
            f'Design system sauvegardé — {len(design_system.get("components", {}))} composants, '
            f'{len(design_system.get("tokens", {}).get("colors", {}))} couleurs.'
        )
        return {'completed': True}

    def _parse_json(self, raw: str) -> dict | None:
        try:
            text = raw.strip()
            if text.startswith('```'):
                lines = text.split('\n')
                text = '\n'.join(lines[1:-1]) if lines[-1].strip() == '```' else '\n'.join(lines[1:])
            return json.loads(text)
        except json.JSONDecodeError:
            return None

    def _display_summary(self, ds: dict) -> None:
        colors = ds.get('tokens', {}).get('colors', {})
        components = ds.get('components', {})
        self.io.print_section('Design System Généré')
        self.io.print(f'  Couleurs     : {len(colors)} tokens ({", ".join(list(colors.keys())[:5])}...)')
        self.io.print(f'  Composants   : {len(components)} ({", ".join(list(components.keys())[:6])}...)')
        primary = colors.get('primary', '?')
        bg = colors.get('background', '?')
        self.io.print(f'  Primary      : {primary}  |  Background : {bg}')
```

---

## Implémentation — `DesignReviewer`

Utilisé par `ExecutionLoop` après chaque tâche UI pour vérifier la cohérence avec le design system.

```python
# src/workflow/tools/design_reviewer.py
import json
from workflow.core.project_memory import ProjectMemory
from workflow.llm.llm_provider import LLMProvider

DESIGN_REVIEW_PROMPT = """Tu es un reviewer UI. Analyse ce code et vérifie s'il respecte le design system.

Design System (tokens) :
{tokens_summary}

Code généré :
{code_diff}

Vérifie :
1. Couleurs hardcodées qui devraient utiliser les tokens (ex: "#2563eb" au lieu de la variable primary)
2. Valeurs de spacing/radius arbitraires qui devraient utiliser la scale définie
3. Composants UI inventés qui existent déjà dans le design system
4. Typographie incohérente avec les font-sizes définis

Retourne une liste JSON d'issues (vide si tout est correct) :
[{{"type": "hardcoded_color|arbitrary_spacing|missing_component|font_mismatch", "location": "fichier:ligne", "message": "..."}}]

Retourne UNIQUEMENT le JSON."""


class DesignReviewer:
    def __init__(self, project_root: str, llm: LLMProvider):
        self.memory = ProjectMemory(project_root)
        self.llm = llm

    async def review(self, generated_files: dict[str, str]) -> list[dict]:
        """Vérifier que le code respecte le design system. Retourne une liste d'issues."""
        design_system = await self.memory.get_design_system()
        if not design_system:
            return []  # Pas de design system → pas de review

        # Résumé compact des tokens pour le prompt
        tokens = design_system.get('tokens', {})
        tokens_summary = json.dumps({
            'colors': tokens.get('colors', {}),
            'spacing': tokens.get('spacing', []),
            'radius': tokens.get('radius', {}),
            'typography_sizes': tokens.get('typography', {}).get('sizes', {}),
        }, indent=2)

        # Concaténer les fichiers générés pour la review
        code_diff = '\n\n'.join(
            f'### {path}\n```\n{content[:1000]}\n```'
            for path, content in generated_files.items()
            if self._is_ui_file(path)
        )

        if not code_diff:
            return []  # Pas de fichier UI → rien à vérifier

        raw = await self.llm.ask(
            DESIGN_REVIEW_PROMPT.format(
                tokens_summary=tokens_summary,
                code_diff=code_diff,
            ),
            role='fast',
        )

        try:
            text = raw.strip()
            if text.startswith('```'):
                lines = text.split('\n')
                text = '\n'.join(lines[1:-1])
            return json.loads(text) or []
        except (json.JSONDecodeError, ValueError):
            return []

    def _is_ui_file(self, path: str) -> bool:
        ui_extensions = ('.tsx', '.jsx', '.vue', '.svelte', '.html', '.css', '.scss', '.py')
        ui_keywords = ('component', 'view', 'page', 'template', 'widget', 'screen', 'ui')
        path_lower = path.lower()
        return (
            any(path_lower.endswith(ext) for ext in ui_extensions)
            or any(kw in path_lower for kw in ui_keywords)
        )
```

---

## Intégration dans `ExecutionLoop`

Ajouter dans `ExecutionLoop.run()` après `_apply_generated_files()` :

```python
# Dans ExecutionLoop.run(), après apply_generated_files()
if hasattr(self, '_design_reviewer'):
    design_issues = await self._design_reviewer.review(applied_contents)
    if design_issues:
        for issue in design_issues:
            self.io.print_warning(f'[Design] {issue["type"]} — {issue["message"]}')
        # Bloquer et corriger si issues critiques (hardcoded colors)
        critical = [i for i in design_issues if i['type'] == 'hardcoded_color']
        if critical and attempt < MAX_RETRIES:
            last_error = {'output': f'Design issues: {json.dumps(critical)}'}
            continue
```

---

## Nouvelles méthodes dans `ProjectMemory`

```python
async def get_design_system(self) -> dict | None: ...
async def save_design_system(self, ds: dict) -> None: ...
async def get_screen_flow(self) -> str | None: ...
async def save_screen_flow(self, content: str) -> None: ...
```

Nouveaux chemins dans `WorkflowPaths` :
```python
@property
def design_system(self) -> Path:
    return self.workflow_dir / 'design-system.json'

@property
def screen_flow(self) -> Path:
    return self.workflow_dir / 'screen-flow.md'
```

---

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `PHASE_ORDER` contient `'DESIGN'` entre `'DISCOVERY'` et `'SPECIFICATION'` | ⬜ |
| 2 | `design-system.json` contient colors, typography, spacing, radius, shadows, components | ⬜ |
| 3 | `screen-flow.md` liste les écrans avec leurs routes et composants principaux | ⬜ |
| 4 | Phase DESIGN proposée de conserver le design system existant si présent | ⬜ |
| 5 | `DesignReviewer.review()` retourne `[]` si aucun fichier UI dans la tâche | ⬜ |
| 6 | `DesignReviewer` utilise `role='fast'` (Haiku) pour la review | ⬜ |
| 7 | Les issues de type `hardcoded_color` bloquent et forcent un retry | ⬜ |
| 8 | `DesignSystemPhase` utilise `role='reasoning'` pour la génération | ⬜ |
| 9 | Les mockups dans `ValidationPhase` référencent les composants du design system | ⬜ |
| 10 | `ProjectMemory` expose `get_design_system()` et `save_design_system()` | ⬜ |
