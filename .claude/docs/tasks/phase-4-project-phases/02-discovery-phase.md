# Phase 2 — Tâche 2.4 : DiscoveryPhase.py + SpecificationPhase.py

## Objectif

Implémenter les deux premières phases du cycle projet :
- `DiscoveryPhase` — dialogue interactif pour capturer la vision et le style de design
- `SpecificationPhase` — génération structurée des fonctionnalités par version

## Dépendances

- Tâches 2.1 ✅, 2.2 ✅, 2.3 ✅
- Phase 1 ✅

## Fichiers à Créer

- `src/workflow/phases/discovery_phase.py` [CRÉER]
- `src/workflow/phases/specification_phase.py` [CRÉER]

---

## DiscoveryPhase

### Responsabilités

1. Poser 3 à 5 questions ciblées sur l'idée initiale (`role='reasoning'`)
2. Collecter le style de design souhaité (choix parmi 10 options)
3. Écrire `vision.md` et `design.json` dans `.workflow/`
4. Retourner `{'completed': True}` quand les deux fichiers sont écrits

### Styles de Design Disponibles

```
minimaliste    | Material Design | Glassmorphism | Néomorphisme
Brutaliste     | Doux/Pastel     | Dashboard Pro | Mobile-First
Cyberpunk      | Personnalisé
```

### Implémentation

```python
# src/workflow/phases/discovery_phase.py
import json
from datetime import datetime, timezone

from workflow.core.project_memory import ProjectMemory
from workflow.llm.llm_provider import LLMProvider
from workflow.llm.prompt_builder import PromptBuilder

DESIGN_STYLES = [
    ('minimaliste',   'Minimaliste',     'Épuré, espaces blancs, typographie'),
    ('material',      'Material Design', 'Ombres, couleurs vives, Google'),
    ('glassmorphism', 'Glassmorphism',   'Translucide, flou, moderne'),
    ('neumorphism',   'Néomorphisme',    'Relief doux, monochrome'),
    ('brutalist',     'Brutaliste',      'Contrastes forts, grilles brisées'),
    ('soft_pastel',   'Doux/Pastel',     'Couleurs douces, arrondi'),
    ('dashboard',     'Dashboard Pro',   'Dense, données, KPIs'),
    ('mobile_first',  'Mobile-First',    'Touch-friendly, navigation bas'),
    ('cyberpunk',     'Cyberpunk',       'Neon, sombre, futuriste'),
    ('custom',        'Personnalisé',    'Décrit par l\'utilisateur'),
]


class DiscoveryPhase:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.memory = ProjectMemory(project_root)
        self.llm = llm
        self.io = io

    async def run(self) -> dict:
        """Exécuter la phase Discovery — retourne {'completed': True} si terminée"""
        project = await self.memory.get_project()
        existing_vision = await self.memory.get_vision()

        initial_text = existing_vision or (project or {}).get('initial_description', '')

        self.io.print_header('Phase Discovery — Exploration du Projet')

        if not initial_text:
            initial_text = self.io.prompt(
                'Décris ton projet en quelques phrases (idée, problème résolu, utilisateurs cibles)'
            )
            if not initial_text.strip():
                self.io.print_warning('Description vide — phase annulée.')
                return {'completed': False}

        self.io.print_info('Analyse de l\'idée en cours...')
        questions_raw = await self.llm.ask(
            PromptBuilder.discovery(initial_text),
            role='reasoning',
        )

        self.io.print_section('Questions de clarification')
        self.io.print(questions_raw)

        self.io.print_section('Tes réponses')
        answers = self.io.prompt(
            'Réponds aux questions ci-dessus (librement, dans n\'importe quel ordre)',
            multiline=True,
        )

        vision_content = self._build_vision_md(initial_text, questions_raw, answers)
        await self.memory.save_vision(vision_content)

        design = await self._collect_design_style()
        await self.memory.save_design(design)

        self.io.print_success('Vision et style de design enregistrés.')
        return {'completed': True}

    def _build_vision_md(self, initial: str, questions: str, answers: str) -> str:
        return f"""# Vision Produit

## Description Initiale

{initial}

## Questions de Clarification

{questions}

## Réponses

{answers}

---
*Généré par DiscoveryPhase le {datetime.now(timezone.utc).strftime('%Y-%m-%d')}*
"""

    async def _collect_design_style(self) -> dict:
        self.io.print_section('Style de Design')
        self.io.print('Quel style visuel veux-tu pour ton application ?\n')

        for i, (key, label, desc) in enumerate(DESIGN_STYLES, 1):
            self.io.print(f'  {i:2d}. {label:<20} — {desc}')

        raw = self.io.prompt('\nChoix (numéro ou nom)')
        style_key, style_label = self._parse_style_choice(raw)

        color_scheme = self.io.prompt(
            'Palette de couleurs (clair / sombre / auto)',
            default='clair',
        )
        references_raw = self.io.prompt(
            'Applications de référence pour le style (ex: Linear, Stripe — optionnel)',
            default='',
        )
        references = [r.strip() for r in references_raw.split(',') if r.strip()]

        custom_notes = None
        if style_key == 'custom':
            custom_notes = self.io.prompt('Décris ton style personnalisé')

        return {
            'style': style_key,
            'styleLabel': style_label,
            'colorScheme': color_scheme,
            'references': references,
            'customNotes': custom_notes,
            'collectedAt': datetime.now(timezone.utc).isoformat(),
        }

    def _parse_style_choice(self, raw: str) -> tuple[str, str]:
        raw = raw.strip()
        if raw.isdigit():
            idx = int(raw) - 1
            if 0 <= idx < len(DESIGN_STYLES):
                key, label, _ = DESIGN_STYLES[idx]
                return key, label
        raw_lower = raw.lower()
        for key, label, _ in DESIGN_STYLES:
            if raw_lower in key.lower() or raw_lower in label.lower():
                return key, label
        return 'minimaliste', 'Minimaliste'
```

---

## SpecificationPhase

### Responsabilités

1. Lire `vision.md` et `features.json` existants
2. Générer les fonctionnalités par version via LLM (`role='reasoning'`)
3. Afficher les fonctionnalités proposées et demander validation/correction
4. Écrire `features.json` validé dans `.workflow/`

### Implémentation

```python
# src/workflow/phases/specification_phase.py
import json
from datetime import datetime, timezone

from workflow.core.project_memory import ProjectMemory
from workflow.llm.llm_provider import LLMProvider
from workflow.llm.prompt_builder import PromptBuilder


class SpecificationPhase:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.memory = ProjectMemory(project_root)
        self.llm = llm
        self.io = io

    async def run(self) -> dict:
        """Exécuter la phase Specification — retourne {'completed': True} si terminée"""
        vision = await self.memory.get_vision()
        if not vision:
            self.io.print_error('Pas de vision.md — lance la phase Discovery d\'abord.')
            return {'completed': False}

        existing_features = await self.memory.get_features()

        self.io.print_header('Phase Specification — Définition des Fonctionnalités')
        self.io.print_info('Génération des fonctionnalités en cours...')

        raw = await self.llm.ask(
            PromptBuilder.spec_features(vision, existing_features),
            role='reasoning',
        )

        features = self._parse_features_json(raw)
        if not features:
            self.io.print_error('Le LLM n\'a pas retourné de JSON valide.')
            self.io.print(raw)
            return {'completed': False}

        self._display_features(features)

        while True:
            choice = self.io.prompt(
                '\n[v]alider | [c]orriger | [r]egénérer',
                default='v',
            ).strip().lower()

            if choice in ('v', 'valider', ''):
                break
            elif choice in ('r', 'regénérer'):
                self.io.print_info('Régénération en cours...')
                raw = await self.llm.ask(
                    PromptBuilder.spec_features(vision, features),
                    role='reasoning',
                )
                features = self._parse_features_json(raw) or features
                self._display_features(features)
            elif choice in ('c', 'corriger'):
                correction = self.io.prompt(
                    'Décris les corrections à apporter'
                )
                corrected_raw = await self.llm.ask(
                    PromptBuilder.spec_features(
                        vision + f'\n\nCorrections demandées : {correction}',
                        features,
                    ),
                    role='reasoning',
                )
                features = self._parse_features_json(corrected_raw) or features
                self._display_features(features)

        await self.memory.save_features(features)
        count = sum(len(v) for v in features.values())
        self.io.print_success(f'Fonctionnalités validées ({count} fonctionnalités enregistrées).')
        return {'completed': True}

    def _parse_features_json(self, raw: str) -> dict | None:
        try:
            text = raw.strip()
            if text.startswith('```'):
                lines = text.split('\n')
                text = '\n'.join(lines[1:-1]) if lines[-1].strip() == '```' else '\n'.join(lines[1:])
            return json.loads(text)
        except json.JSONDecodeError:
            return None

    def _display_features(self, features: dict) -> None:
        priority_icon = {'HIGH': '[HIGH]', 'MEDIUM': '[MED] ', 'LOW': '[LOW] '}
        for version, feature_list in features.items():
            self.io.print_section(f'Version {version}')
            for f in feature_list:
                icon = priority_icon.get(f.get('priority', ''), '[    ]')
                self.io.print(
                    f"  {icon} [{f.get('id', '?')}] {f.get('name', '')} — {f.get('description', '')}"
                )
                if f.get('intent'):
                    self.io.print(f"         Intent : {f['intent']}")
```

---

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `DiscoveryPhase.run()` écrit `vision.md` ET `design.json` avant `completed: True` | ⬜ |
| 2 | `_collect_design_style()` accepte numéro OU nom de style | ⬜ |
| 3 | Style `custom` déclenche une question supplémentaire pour `customNotes` | ⬜ |
| 4 | `design.json` contient `style`, `styleLabel`, `colorScheme`, `references`, `customNotes`, `collectedAt` | ⬜ |
| 5 | `DiscoveryPhase` utilise `role='reasoning'` pour l'appel LLM | ⬜ |
| 6 | `SpecificationPhase.run()` retourne `completed: False` si `vision.md` absent | ⬜ |
| 7 | La boucle de validation accepte `v` / `c` / `r` et leurs variantes longues | ⬜ |
| 8 | `_parse_features_json()` gère les blocs markdown (```json ... ```) autour du JSON | ⬜ |
| 9 | `SpecificationPhase` utilise `role='reasoning'` pour tous les appels LLM | ⬜ |
| 10 | `features.json` respecte la structure `{"v1.0": [...], "v1.5": [...]}` | ⬜ |
