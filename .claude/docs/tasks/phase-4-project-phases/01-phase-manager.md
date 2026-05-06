# Phase 4 — Tâche 4.1 : PhaseManager.py (revisitable, pas waterfall)

## Objectif

Créer le `PhaseManager.py` — l'orchestrateur des 5 phases projet (Discovery, Specification, Design, Architecture, Validation, Active). **Ces phases sont revisitables** : l'utilisateur peut revenir à une phase antérieure à tout moment, ou démarrer en `Starter Mode` qui saute directement à une tâche concrète.

> **Reframe critique vs design initial** : on n'est PAS dans un tunnel waterfall. La vision émerge en codant, les fonctionnalités évoluent, l'architecture se précise. PhaseManager doit supporter ce mouvement, pas le contraindre.

## Dépendances

- Tâches 2.1 ✅, 2.2 ✅, 2.3 ✅
- Phase 1 ✅

## Fichiers à Créer

- `src/workflow/core/phase_manager.py` [CRÉER]

## Modes d'Existence

```
┌─ STARTER MODE ──────────────────────────────────────────────┐
│  workflow start --quick "ajoute un endpoint /users"        │
│  → Génère vision minimale + tech-stack auto-détecté        │
│  → Crée TASK-001 directement (atteint phase ACTIVE)        │
│  → En arrière-plan : Discovery + Spec + Arch en background │
└─────────────────────────────────────────────────────────────┘

┌─ FULL MODE ─────────────────────────────────────────────────┐
│  workflow start --full                                      │
│  → DISCOVERY → SPECIFICATION → DESIGN → ARCHITECTURE        │
│  → → VALIDATION → ACTIVE (revisitable à tout moment)       │
└─────────────────────────────────────────────────────────────┘
```

## Ordre de Phase (avec DESIGN inséré)

```python
PHASE_ORDER = [
    'DISCOVERY',         # Vision libre
    'SPECIFICATION',     # Features.json
    'DESIGN',            # Style + design.json (NOUVEAU dans l'ordre)
    'ARCHITECTURE',      # tech-stack.json + TASK-001/002
    'VALIDATION',        # Tâches détaillées (utilise design + tech-stack)
    'ACTIVE',            # ExecutionPhase
]
```

> **Ordre critique** : ARCHITECTURE avant VALIDATION (tech-stack nécessaire). DESIGN avant ARCHITECTURE (le style influence parfois la stack — PWA vs native, par ex). DESIGN après SPECIFICATION (besoin des features pour générer screen-flow).

## Implémentation

```python
# src/workflow/core/phase_manager.py
from workflow.core.project_memory import ProjectMemory
from workflow.phases.discovery_phase import DiscoveryPhase
from workflow.phases.specification_phase import SpecificationPhase
from workflow.phases.design_system_phase import DesignSystemPhase
from workflow.phases.architecture_phase import ArchitecturePhase
from workflow.phases.validation_phase import ValidationPhase
from workflow.phases.execution_phase import ExecutionPhase
from workflow.llm.llm_provider import LLMProvider

PHASE_ORDER = [
    'DISCOVERY',
    'SPECIFICATION',
    'DESIGN',
    'ARCHITECTURE',
    'VALIDATION',
    'ACTIVE',
]


class PhaseManager:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.memory = ProjectMemory(project_root)
        self.llm = llm
        self.io = io

        self.phases = {
            'DISCOVERY': DiscoveryPhase(project_root, llm, io),
            'SPECIFICATION': SpecificationPhase(project_root, llm, io),
            'DESIGN': DesignSystemPhase(project_root, llm, io),
            'ARCHITECTURE': ArchitecturePhase(project_root, llm, io),
            'VALIDATION': ValidationPhase(project_root, llm, io),
            'ACTIVE': ExecutionPhase(project_root, llm, io),
        }

    # ─── Lecture d'état ───────────────────────────────────────────────

    async def get_current_phase(self) -> str:
        project = await self.memory.get_project()
        return (project or {}).get('status', 'DISCOVERY')

    # ─── Avance / Recul / Saut ────────────────────────────────────────

    async def advance_phase(self) -> str:
        """Avancer à la phase suivante (linéaire)"""
        current = await self.get_current_phase()
        idx = PHASE_ORDER.index(current) if current in PHASE_ORDER else -1
        if idx < len(PHASE_ORDER) - 1:
            next_phase = PHASE_ORDER[idx + 1]
            await self.memory.update_project({'status': next_phase})
            return next_phase
        return current

    async def revisit_phase(self, phase: str, reason: str = '') -> dict:
        """
        Revenir à une phase antérieure (ex : ajouter une fonctionnalité après avoir codé).
        Ne perd RIEN — les artéfacts existants (vision.md, features.json...) sont conservés
        et seront amendés. Annote dans le journal projet la raison du retour.
        """
        if phase not in PHASE_ORDER:
            raise ValueError(f'Phase inconnue : {phase}')
        current = await self.get_current_phase()
        await self.memory.update_project({'status': phase})
        await self.memory.append_phase_history({
            'from': current,
            'to': phase,
            'kind': 'REVISIT',
            'reason': reason,
        })
        self.io.print_info(f'Retour à la phase {phase} — {reason}')
        return {'previous': current, 'now': phase}

    async def jump_to_active(self, starter_seed: dict) -> dict:
        """
        Starter Mode — sauter directement à ACTIVE en générant le minimum.
        starter_seed = { 'description': str, 'auto_detect_stack': bool }
        """
        # 1. Générer une vision minimale
        await self.memory.set_vision(
            f"# Vision (Starter Mode)\n\n{starter_seed['description']}\n\n"
            f"_Cette vision a été générée en Starter Mode. À enrichir via "
            f"`workflow start --revisit DISCOVERY`._"
        )

        # 2. Auto-détecter la stack si demandé
        if starter_seed.get('auto_detect_stack'):
            stack = await self._detect_stack_from_repo()
            if stack:
                await self.memory.set_tech_stack(stack)

        # 3. Créer une feature minimale et une tâche
        await self.memory.set_features({'v1.0': [{
            'id': 'F001',
            'name': 'Tâche initiale',
            'description': starter_seed['description'],
            'priority': 'HIGH',
            'intent': 'Démarrage Starter Mode',
        }]})

        # 4. Sauter à ACTIVE
        await self.memory.update_project({'status': 'ACTIVE'})
        await self.memory.append_phase_history({
            'from': 'DISCOVERY',
            'to': 'ACTIVE',
            'kind': 'STARTER_JUMP',
            'reason': starter_seed['description'][:100],
        })

        # 5. Lancer Discovery + Specification en background (non bloquant)
        # → Géré par WorkflowAgent.start_background_phases()

        return {'mode': 'STARTER', 'now': 'ACTIVE'}

    async def _detect_stack_from_repo(self) -> dict | None:
        """Détecter la stack depuis les fichiers existants (package.json, pyproject.toml, etc.)"""
        from pathlib import Path
        root = Path(self.memory.project_root)
        if (root / 'pyproject.toml').exists():
            return {
                'language': 'python',
                'framework': 'detected-from-pyproject',
                'build_validate': 'uv run ruff check .',
                'test': 'uv run pytest',
                'allowed_commands': ['uv run pytest', 'uv run ruff check .'],
            }
        if (root / 'package.json').exists():
            return {
                'language': 'javascript',
                'framework': 'detected-from-package-json',
                'build_validate': 'npm run lint',
                'test': 'npm test',
                'allowed_commands': ['npm run lint', 'npm test'],
            }
        if (root / 'Cargo.toml').exists():
            return {
                'language': 'rust',
                'framework': 'cargo',
                'build_validate': 'cargo clippy',
                'test': 'cargo test',
                'allowed_commands': ['cargo clippy', 'cargo test', 'cargo build'],
            }
        return None

    # ─── Exécution ────────────────────────────────────────────────────

    async def run_current_phase(self) -> dict:
        """Exécuter la phase courante"""
        phase = await self.get_current_phase()
        handler = self.phases.get(phase)
        if not handler:
            raise ValueError(f'Phase inconnue : {phase}')

        result = await handler.run()

        # Avancer automatiquement seulement si la phase n'a pas demandé un revisit
        if result.get('completed') and not result.get('revisit'):
            await self.advance_phase()
        elif result.get('revisit'):
            await self.revisit_phase(result['revisit'], reason=result.get('reason', ''))

        return result

    # ─── Validation ───────────────────────────────────────────────────

    async def can_start_phase(self, phase: str) -> bool:
        """
        Vérifier les prérequis pour démarrer une phase.
        Mode revisit toujours autorisé. Mode forward : vérifier sortie de la phase précédente.
        """
        if phase not in PHASE_ORDER:
            return False

        # Une phase revisitable est toujours démarrable si on a déjà été plus loin
        current = await self.get_current_phase()
        if current in PHASE_ORDER and PHASE_ORDER.index(current) > PHASE_ORDER.index(phase):
            return True  # Revisit OK

        # Forward : prérequis spécifiques par phase
        return await self._check_forward_prereqs(phase)

    async def _check_forward_prereqs(self, phase: str) -> bool:
        if phase == 'DISCOVERY':
            return True  # Toujours démarrable
        if phase == 'SPECIFICATION':
            return bool(await self.memory.get_vision())
        if phase == 'DESIGN':
            return bool(await self.memory.get_features())
        if phase == 'ARCHITECTURE':
            return bool(await self.memory.get_features())  # design optionnel
        if phase == 'VALIDATION':
            return bool(await self.memory.get_tech_stack())
        if phase == 'ACTIVE':
            project = await self.memory.get_project()
            return bool((project or {}).get('active_version'))
        return False
```

## Intégration dans `WorkflowAgent` (Phase 5)

```python
# src/workflow/core/workflow_agent.py

async def start(self, mode: str = 'full', description: str | None = None):
    """
    mode = 'full' : phases linéaires Discovery → Active
    mode = 'quick' : Starter Mode, saut direct à ACTIVE
    """
    pm = self._get_phase_manager()

    if mode == 'quick':
        if not description:
            raise ValueError('--quick nécessite une description')
        await pm.jump_to_active({
            'description': description,
            'auto_detect_stack': True,
        })
        # Lancer en background : enrichir Discovery / Spec / Arch sans bloquer
        asyncio.create_task(self._enrich_background())
        await pm.run_current_phase()  # ACTIVE
    else:
        await pm.run_current_phase()
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `PHASE_ORDER` inclut DESIGN entre SPECIFICATION et ARCHITECTURE | ⬜ |
| 2 | `get_current_phase()` lit le statut depuis `project.json` | ⬜ |
| 3 | `advance_phase()` avance linéairement, ne dépasse pas ACTIVE | ⬜ |
| 4 | `revisit_phase('DISCOVERY', reason='...')` change l'état ET annote dans phase_history | ⬜ |
| 5 | `revisit_phase()` ne perd PAS les artéfacts existants (vision.md reste) | ⬜ |
| 6 | `jump_to_active(seed)` génère vision minimale + features minimale + saute à ACTIVE | ⬜ |
| 7 | `jump_to_active()` détecte automatiquement la stack si pyproject.toml/package.json/Cargo.toml | ⬜ |
| 8 | `can_start_phase('DISCOVERY')` retourne True si on est en SPECIFICATION (revisit OK) | ⬜ |
| 9 | `can_start_phase('VALIDATION')` retourne False sans tech-stack.json | ⬜ |
| 10 | `run_current_phase()` respecte le signal `revisit` retourné par une phase | ⬜ |
| 11 | Tests : starter mode complet (mock auto-detect), revisit Discovery après ACTIVE | ⬜ |

## Notes d'Implémentation

- Le Starter Mode n'est pas un mode dégradé — c'est le **mode dominant** pour un freelance qui ouvre un repo existant et veut commencer à coder. Le mode full est pour les greenfield ambitieux.
- `phase_history` (dans `project.json`) trace tous les mouvements (forward, revisit, starter_jump). Utile pour `workflow audit` et debugging.
- Si une phase est revisitée, son module (ex : `DiscoveryPhase`) doit savoir lire l'état existant et proposer un *amendement* — pas refaire tout depuis zéro.
