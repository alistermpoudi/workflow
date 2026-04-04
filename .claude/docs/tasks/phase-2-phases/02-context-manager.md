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

## Scoring de Pertinence

Le `ContextManager` ne charge pas les décisions de façon mécanique (ex: "les 5 dernières"). Il leur attribue un **score de pertinence** par rapport à la tâche courante. Seules les décisions avec score > seuil sont incluses dans le contexte LLM.

```
Score(décision, tâche) =
  similarité_mots_clés(décision.summary, tâche.title + tâche.intent)  [0–1]
  × poids_récence(décision.date)           [0.5 | 0.8 | 1.0]
  × poids_scope(décision.scope, tâche)     [0.3 | 0.8 | 1.0]
    — global → 1.0
    — scope local présent dans tâche.context → 0.8
    — scope local absent de tâche.context → 0.3
  × (décision.confidence === 'HIGH' ? 1.0 : 0.7)
```

**Budget tokens** : Le `ContextManager` maintient un budget (défaut : 2000 tokens pour les décisions). Les décisions sont triées par score décroissant et chargées jusqu'à épuisement du budget. Idem pour les fichiers source.

**Seuil** : Score minimum `0.4` — en dessous, la décision n'est pas chargée même si le budget le permet.

Cette approche remplace le keyword matching statique de `DecisionsLog.getRelevant()` pour le niveau 3.

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

    // Décisions scorées par pertinence (remplace le keyword matching statique)
    const relevantDecisions = await this._loadScoredDecisions(task);

    return {
      task,
      relevantFiles,
      relevantDecisions,
    };
  }

  // Charger les décisions avec scoring de pertinence
  // Dépend de DecisionsLog.getAll() — méthode qui retourne toutes les entrées parsées
  async _loadScoredDecisions(task, options = {}) {
    const tokenBudget = options.tokenBudget ?? 2000;
    const minScore = options.minScore ?? 0.4;

    const allDecisions = await this.decisions.getAll();
    if (!allDecisions.length) return [];

    // Scorer chaque décision
    const scored = allDecisions.map(d => ({
      ...d,
      score: this._scoreDecision(d, task),
    }));

    // Trier par score décroissant, filtrer sous le seuil
    const filtered = scored
      .filter(d => d.score >= minScore)
      .sort((a, b) => b.score - a.score);

    // Appliquer le budget tokens (approximation : ~4 chars/token)
    const result = [];
    let usedTokens = 0;
    for (const d of filtered) {
      const tokens = Math.ceil(d.raw.length / 4);
      if (usedTokens + tokens > tokenBudget) break;
      result.push(d);
      usedTokens += tokens;
    }
    return result;
  }

  // Calculer le score de pertinence d'une décision par rapport à une tâche
  // Formule : similarity × recencyWeight × scopeWeight × confidenceWeight
  _scoreDecision(decision, task) {
    const searchText = `${task.title} ${task.intent ?? ''} ${task.context ?? ''}`.toLowerCase();
    const decisionText = decision.summary.toLowerCase();

    // Similarité mots-clés (Jaccard simplifié)
    const taskWords = new Set(searchText.split(/\W+/).filter(w => w.length > 3));
    const decWords = new Set(decisionText.split(/\W+/).filter(w => w.length > 3));
    const intersection = [...taskWords].filter(w => decWords.has(w)).length;
    const union = new Set([...taskWords, ...decWords]).size;
    const similarity = union > 0 ? intersection / union : 0;

    // Récence (décisions < 7 jours = 1.0, < 30 jours = 0.8, plus ancien = 0.5)
    const ageMs = Date.now() - new Date(decision.date).getTime();
    const ageDays = ageMs / (1000 * 60 * 60 * 24);
    const recencyWeight = ageDays < 7 ? 1.0 : ageDays < 30 ? 0.8 : 0.5;

    // Scope : une décision "global" s'applique à tout, une décision "local" seulement si
    // la tâche est dans le même module
    const scopeWeight = !decision.scope || decision.scope === 'global' ? 1.0
      : task.context?.toLowerCase().includes(decision.scope.toLowerCase()) ? 0.8
      : 0.3;

    // Confiance
    const confidenceWeight = decision.confidence === 'HIGH' ? 1.0 : 0.7;

    return similarity * recencyWeight * scopeWeight * confidenceWeight;
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
| 6 | `_scoreDecision()` retourne un score plus élevé pour une décision avec mots-clés communs | ⬜ |
| 6b | `_scoreDecision()` applique `scopeWeight=1.0` pour scope "global", `0.3` pour scope local non-correspondant | ⬜ |
| 7 | `_loadScoredDecisions()` respecte le budget tokens (ne dépasse pas 2000 tokens) | ⬜ |
| 8 | `_loadScoredDecisions()` exclut les décisions avec score < 0.4 | ⬜ |
| 9 | Tests unitaires vérifient que les niveaux sont chargés indépendamment | ⬜ |
