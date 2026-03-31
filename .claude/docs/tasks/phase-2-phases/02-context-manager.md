# Phase 2 — Tâche 2.2 : ContextManager.js

## Objectif

Créer le `ContextManager.js` qui implémente la hiérarchie de chargement stricte du contexte LLM. C'est ce composant qui garantit que Workflow ne sature jamais le contexte en chargeant tout le projet d'un coup.

## Dépendances

- Tâche 2.1 ✅ (`LLMProvider`, `PromptBuilder`)
- Phase 1 ✅

## Fichiers à Créer

- `src/core/ContextManager.js` [CRÉER]
- `tests/unit/ContextManager.test.js` [CRÉER]

## Règle Fondamentale

```
Niveau 1 — Système     : toujours chargé (~500 tokens max)
Niveau 2 — Version     : chargé au switch de version
Niveau 3 — Tâche       : chargé au start task
Niveau 4 — On-demand   : uniquement si nécessaire (CodeIndexer — Phase 6)
```

**Ne jamais charger le niveau 4 en "contexte de base". Ne jamais tout charger d'un coup.**

## Implémentation

```javascript
// src/core/ContextManager.js
import { ProjectMemory } from './ProjectMemory.js';
import { TaskManager } from '../tools/TaskManager.js';
import { DecisionsLog } from './DecisionsLog.js';
import { FileSystem } from '../tools/FileSystem.js';

export class ContextManager {
  constructor(projectRoot) {
    this.memory = new ProjectMemory(projectRoot);
    this.tasks = new TaskManager(projectRoot);
    this.decisions = new DecisionsLog(projectRoot);
    this.fs = new FileSystem(projectRoot);

    // Cache en mémoire pour éviter les relectures répétées
    this._systemContext = null;
    this._versionContext = null;
    this._activeVersion = null;
  }

  // Niveau 1 — Contexte système (toujours chargé, mis en cache)
  async getSystemContext() {
    if (this._systemContext) return this._systemContext;

    const summary = await this.memory.getProjectSummary();
    const techStack = await this.memory.getTechStack();

    this._systemContext = {
      project: summary,
      techStack: techStack ? {
        language: techStack.language,
        framework: techStack.framework,
        database: techStack.database,
        // Exclure build_validate et allowed_commands du contexte système
        // (trop verbeux, chargés uniquement quand ExecutionLoop en a besoin)
      } : null,
    };

    return this._systemContext;
  }

  // Niveau 2 — Contexte version (chargé au switch)
  async getVersionContext(version) {
    if (this._activeVersion === version && this._versionContext) {
      return this._versionContext;
    }

    const meta = await this.memory.getVersionMeta(version);
    const progress = await this.memory.getProgress(version);

    this._versionContext = {
      meta,
      // Seulement les IDs, pas le contenu des tâches
      doneTasks: progress.done,
      pendingTasks: progress.pending,
      failedTasks: progress.failed ?? [],
      deferredTasks: progress.deferred ?? [],
    };
    this._activeVersion = version;

    return this._versionContext;
  }

  // Niveau 3 — Contexte tâche (chargé au start task)
  async getTaskContext(version, taskId) {
    const task = await this.tasks.getTask(version, taskId);
    if (!task) throw new Error(`Tâche ${taskId} introuvable dans la version ${version}`);

    // Lire sélectivement les fichiers mentionnés dans la tâche
    const relevantFiles = await this.fs.readSelective(task.filesToModify ?? []);

    // 5 dernières décisions pertinentes
    const relevantDecisions = await this.decisions.getRelevant(task);

    return {
      task,
      relevantFiles,
      relevantDecisions,
    };
  }

  // Construire le contexte complet pour un appel LLM
  // Charge uniquement les niveaux nécessaires
  async buildLLMContext(options = {}) {
    const ctx = {};

    ctx.system = await this.getSystemContext();

    if (options.version) {
      ctx.version = await this.getVersionContext(options.version);
    }

    if (options.taskId && options.version) {
      ctx.task = await this.getTaskContext(options.version, options.taskId);
    }

    return ctx;
  }

  // Invalider le cache (après un switch de version ou update projet)
  invalidateCache() {
    this._systemContext = null;
    this._versionContext = null;
    this._activeVersion = null;
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `getSystemContext()` met en cache et ne relit pas le fichier deux fois | ⬜ |
| 2 | `getVersionContext()` ne charge pas le contenu des tâches, uniquement les IDs | ⬜ |
| 3 | `getTaskContext()` lit sélectivement les fichiers listés dans la tâche | ⬜ |
| 4 | `buildLLMContext({ version })` ne charge pas le niveau tâche si pas demandé | ⬜ |
| 5 | `invalidateCache()` force un rechargement complet | ⬜ |
| 6 | Tests unitaires vérifient que les niveaux sont chargés indépendamment | ⬜ |
