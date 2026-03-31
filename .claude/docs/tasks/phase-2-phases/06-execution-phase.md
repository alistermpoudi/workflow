# Phase 2 — Tâche 2.6 : ExecutionPhase.js

## Objectif

Créer `ExecutionPhase.js` — l'orchestrateur de la phase de réalisation. Il prend la prochaine tâche en attente, charge le contexte approprié, délègue l'exécution à `ExecutionLoop` (Phase 3), et met à jour l'état.

Note : `ExecutionPhase` dépend de `ExecutionLoop` qui n'existe pas encore. Cette tâche implémente la structure et les méthodes de coordination. L'intégration complète se fait en Phase 3.

## Dépendances

- Tâches 2.1 à 2.5 ✅

## Fichiers à Créer

- `src/phases/ExecutionPhase.js` [CRÉER]

## Implémentation

```javascript
// src/phases/ExecutionPhase.js
import { ProjectMemory } from '../core/ProjectMemory.js';
import { TaskManager } from '../tools/TaskManager.js';
import { ContextManager } from '../core/ContextManager.js';
import { DecisionsLog } from '../core/DecisionsLog.js';

export class ExecutionPhase {
  constructor(projectRoot, llm, io) {
    this.memory = new ProjectMemory(projectRoot);
    this.tasks = new TaskManager(projectRoot);
    this.context = new ContextManager(projectRoot);
    this.decisions = new DecisionsLog(projectRoot);
    this.llm = llm;
    this.io = io;
    this.executionLoop = null; // Injecté en Phase 3
  }

  // Injecter ExecutionLoop (Phase 3)
  setExecutionLoop(loop) {
    this.executionLoop = loop;
  }

  async run() {
    const activeVersion = await this.memory.getActiveVersion();
    if (!activeVersion) {
      this.io.display('Aucune version active. Lance `workflow version create v1.0 "..."` d\'abord.');
      return { completed: false };
    }

    const nextTask = await this.tasks.getNextTask(activeVersion);
    if (!nextTask) {
      this.io.display(`✅ Toutes les tâches de ${activeVersion} sont terminées !`);
      return { completed: true, versionDone: true };
    }

    return this.executeTask(activeVersion, nextTask);
  }

  async executeTask(version, task) {
    this.io.display(`\n🔄 Tâche en cours : ${task.id} — ${task.title}`);

    // Marquer en cours
    await this.tasks.markInProgress(version, task.id);

    // Charger le contexte de niveau 3
    const ctx = await this.context.buildLLMContext({ version, taskId: task.id });

    // Consulter decisions.log avant de coder
    if (ctx.task.relevantDecisions.length > 0) {
      this.io.display('\n📋 Décisions antérieures pertinentes :');
      ctx.task.relevantDecisions.forEach(d => this.io.display(`  • ${d.summary}`));
    }

    // Déléguer à ExecutionLoop (Phase 3)
    if (!this.executionLoop) {
      // Stub pour Phase 2 — sera remplacé en Phase 3
      this.io.display('[ExecutionLoop non encore implémenté — Phase 3]');
      return { completed: false };
    }

    const result = await this.executionLoop.run(task, ctx);

    if (result.success) {
      await this.tasks.markDone(version, task.id);

      // Enregistrer les décisions prises pendant la tâche
      if (result.decisions?.length > 0) {
        await this.decisions.logMany(task.id, result.decisions);
      }

      this.io.display(`✅ ${task.id} terminée.`);
    } else if (result.escalate) {
      this.io.display(`\n❌ Échec persistant sur ${task.id} après 3 tentatives.`);
      this.io.display(`Contexte d'erreur :\n${result.escalationContext}`);
      const action = await this.io.ask('Que faire ? (skip / retry / split / abort)');

      switch (action.trim().toLowerCase()) {
        case 'skip':
          await this.tasks.deferTask(version, task.id, 'next', 'Sauté manuellement');
          break;
        case 'split':
          this.io.display('La tâche sera découpée en sous-tâches à la prochaine session.');
          break;
        default:
          this.io.display('Session interrompue.');
          return { completed: false };
      }
    }

    return { completed: false }; // La phase n'est jamais "completed" — elle continue jusqu'à versionDone
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `run()` prend la prochaine tâche en attente automatiquement | ⬜ |
| 2 | `run()` affiche les décisions pertinentes avant d'exécuter | ⬜ |
| 3 | `run()` gère les 4 actions d'escalade (skip/retry/split/abort) | ⬜ |
| 4 | Les décisions retournées par `ExecutionLoop` sont enregistrées dans `decisions.log` | ⬜ |
| 5 | `run()` retourne `{ completed: true, versionDone: true }` si plus aucune tâche | ⬜ |
