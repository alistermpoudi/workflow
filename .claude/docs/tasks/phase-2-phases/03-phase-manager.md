# Phase 2 — Tâche 2.3 : PhaseManager.js

## Objectif

Créer le `PhaseManager.js` — l'orchestrateur des 5 phases. Il gère les transitions entre phases, persiste l'état courant dans `project.json`, et garantit que les phases sont exécutées dans le bon ordre.

## Dépendances

- Tâche 2.1 ✅, 2.2 ✅
- Phase 1 ✅

## Fichiers à Créer

- `src/core/PhaseManager.js` [CRÉER]

## États de Phase

```
DISCOVERY → SPECIFICATION → VALIDATION → ARCHITECTURE → ACTIVE
```

`ACTIVE` = Phase 5 Réalisation en cours. Les phases 1-4 ne se rejouent pas une fois `ACTIVE` — uniquement via `workflow version add-task` pour ajouter des tâches.

## Implémentation

```javascript
// src/core/PhaseManager.js
import { ProjectMemory } from './ProjectMemory.js';
import { DiscoveryPhase } from '../phases/DiscoveryPhase.js';
import { SpecificationPhase } from '../phases/SpecificationPhase.js';
import { ValidationPhase } from '../phases/ValidationPhase.js';
import { ArchitecturePhase } from '../phases/ArchitecturePhase.js';
import { ExecutionPhase } from '../phases/ExecutionPhase.js';

const PHASE_ORDER = ['DISCOVERY', 'SPECIFICATION', 'VALIDATION', 'ARCHITECTURE', 'ACTIVE'];

export class PhaseManager {
  constructor(projectRoot, llmProvider, io) {
    this.memory = new ProjectMemory(projectRoot);
    this.llm = llmProvider;
    this.io = io; // Interface I/O (CLI ou MCP)

    this.phases = {
      DISCOVERY: new DiscoveryPhase(projectRoot, llmProvider, io),
      SPECIFICATION: new SpecificationPhase(projectRoot, llmProvider, io),
      VALIDATION: new ValidationPhase(projectRoot, llmProvider, io),
      ARCHITECTURE: new ArchitecturePhase(projectRoot, llmProvider, io),
      ACTIVE: new ExecutionPhase(projectRoot, llmProvider, io),
    };
  }

  // Obtenir la phase courante
  async getCurrentPhase() {
    const project = await this.memory.getProject();
    return project?.status ?? 'DISCOVERY';
  }

  // Avancer à la phase suivante
  async advancePhase() {
    const current = await this.getCurrentPhase();
    const idx = PHASE_ORDER.indexOf(current);
    if (idx < PHASE_ORDER.length - 1) {
      const next = PHASE_ORDER[idx + 1];
      await this.memory.updateProject({ status: next });
      return next;
    }
    return current; // Déjà à ACTIVE
  }

  // Exécuter la phase courante
  async runCurrentPhase() {
    const phase = await this.getCurrentPhase();
    const handler = this.phases[phase];
    if (!handler) throw new Error(`Phase inconnue : ${phase}`);

    const result = await handler.run();

    if (result.completed) {
      await this.advancePhase();
    }

    return result;
  }

  // Vérifier si une phase peut démarrer (ses prérequis sont satisfaits)
  async canStartPhase(phase) {
    const project = await this.memory.getProject();
    const current = project?.status;
    const currentIdx = PHASE_ORDER.indexOf(current);
    const targetIdx = PHASE_ORDER.indexOf(phase);
    return targetIdx <= currentIdx + 1;
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `getCurrentPhase()` lit le statut depuis `project.json` | ⬜ |
| 2 | `advancePhase()` ne peut pas aller au-delà de `ACTIVE` | ⬜ |
| 3 | `runCurrentPhase()` avance automatiquement si `result.completed` | ⬜ |
| 4 | `canStartPhase('ARCHITECTURE')` retourne false si on est en DISCOVERY | ⬜ |
