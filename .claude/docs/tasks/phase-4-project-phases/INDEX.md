# Phase 4 — Les 5 Phases Projet (revisitables)

## Objectif

Implémenter les phases qui transforment une idée en fichiers `.workflow/` prêts à être exécutés. **Reframe critique vs design initial** : ces phases sont **revisitables** — l'utilisateur peut revenir à Discovery après avoir codé pour ajouter une feature, ou démarrer en **Starter Mode** qui saute directement à ACTIVE en générant le minimum.

À l'issue de cette phase, Workflow peut accompagner un utilisateur de "j'ai une idée" jusqu'à "j'ai des tâches en cours d'exécution", sans imposer 1h de Q/R avant la première ligne de code.

## Piliers Adressés

- Support du flow utilisateur autour des **Piliers 1, 3, 4, 5** déjà construits

## Ordre des Phases (avec DESIGN inséré)

```
DISCOVERY → SPECIFICATION → DESIGN → ARCHITECTURE → VALIDATION → ACTIVE
```

> DESIGN s'insère après SPECIFICATION (besoin des features pour générer screen-flow) et avant ARCHITECTURE (le style influence parfois la stack — PWA vs native).

## Stack Phase 4

```
LLMProvider          role='reasoning' pour Discovery/Spec/Arch
                     role='fast' pour scoring, validation
typer + rich         Interface I/O interactive
DecisionsLog         Phase 1 (consultation active)
DesignSystem         génère design.json + screen-flow.md
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 4.1 | [01-phase-manager.md](01-phase-manager.md) | **`PhaseManager.py`** — orchestration des 6 phases, **revisitables**, **Starter Mode**, history |
| 4.2 | [02-discovery-phase.md](02-discovery-phase.md) | `DiscoveryPhase.py` + `SpecificationPhase.py` — vision.md + features.json |
| 4.3 | [04-design-system-phase.md](04-design-system-phase.md) | `DesignSystemPhase.py` + `DesignReviewer.py` — design.json + screen-flow.md |
| 4.4 | [05-architecture-phase.md](05-architecture-phase.md) | `ArchitecturePhase.py` + `ValidationPhase.py` — tech-stack.json + tâches détaillées |
| 4.5 | [07-execution-phase.md](07-execution-phase.md) | `ExecutionPhase.py` — orchestre une tâche (sélection + délégation à `ExecutionLoop`) |

> **Note** : `02-discovery-phase.md` et `05-architecture-phase.md` couvrent chacun deux phases connexes. Les fichiers seront splittés (`03-specification-phase.md`, `06-validation-phase.md`) lorsque le code l'exigera — pour le moment, la duplication serait du papier.

## Dépendances

- Phase 1 complète ✅
- Phase 2 complète ✅
- Phase 3 (`ExecutionLoop`) câblé via `set_execution_loop()` après injection

## Critères de Sortie de Phase

- [ ] `PHASE_ORDER` inclut DESIGN entre SPECIFICATION et ARCHITECTURE
- [ ] `revisit_phase('DISCOVERY', reason='...')` change l'état ET annote dans `phase_history`, sans perdre les artéfacts
- [ ] `jump_to_active(seed)` génère vision/features minimales + saute à ACTIVE
- [ ] `jump_to_active()` détecte automatiquement la stack si `pyproject.toml`/`package.json`/`Cargo.toml`
- [ ] Workflow accompagne un utilisateur de "j'ai une idée" jusqu'aux tâches générées
- [ ] Les tâches générées respectent l'atomicité sémantique (1 tâche = 1 PR mergeable)
- [ ] TASK-001 (linter) et TASK-002 (tests) sont toujours les premières tâches d'une v1.0
- [ ] `DesignSystemPhase` génère `design-system.json` + `screen-flow.md` depuis le style Discovery
- [ ] `ValidationPhase` injecte les mockups ASCII conformes au design dans chaque tâche UI
- [ ] `ExecutionPhase` sélectionne la prochaine tâche prête (dépendances satisfaites)
- [ ] `can_start_phase()` autorise toujours le retour à une phase antérieure (revisit)
