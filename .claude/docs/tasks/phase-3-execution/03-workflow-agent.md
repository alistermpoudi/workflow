# Phase 3 — Tâche 3.3 : WorkflowAgent.js

## Objectif

Créer `WorkflowAgent.js` — l'orchestrateur principal qui relie toutes les pièces ensemble. Il gère le démarrage de session (SyncChecker), délègue aux phases via PhaseManager, et expose les commandes CLI.

## Dépendances

- Toutes les phases 1-3 précédentes ✅

## Fichiers à Créer

- `src/core/WorkflowAgent.js` [CRÉER]

## Implémentation

```javascript
// src/core/WorkflowAgent.js
import chalk from 'chalk';
import { ProjectMemory } from './ProjectMemory.js';
import { PhaseManager } from './PhaseManager.js';
import { SyncChecker } from './SyncChecker.js';
import { VersionManager } from './VersionManager.js';
import { ContextManager } from './ContextManager.js';
import { LLMProvider } from '../llm/LLMProvider.js';
import { ExecutionLoop } from '../tools/ExecutionLoop.js';
// WatchMode et DaemonHeartbeat disponibles en Phase 3 (tâches 3.5 et 3.4)
// import { WatchMode } from '../tools/WatchMode.js';
// import { DaemonHeartbeat } from './DaemonHeartbeat.js';
// OnboardingManager disponible en Phase 6 (v1.5)
// import { OnboardingManager } from './OnboardingManager.js';

export class WorkflowAgent {
  constructor(projectRoot, io) {
    this.projectRoot = projectRoot;
    this.io = io;
    this.memory = new ProjectMemory(projectRoot);
    this.sync = new SyncChecker(projectRoot);
    this.llm = new LLMProvider();
    this.context = new ContextManager(projectRoot);
    this.phaseManager = new PhaseManager(projectRoot, this.llm, io);
    this.versionManager = new VersionManager(projectRoot, io); // Phase 5

    // Namespaces exposés à MCPServer — délèguent à PhaseManager et VersionManager
    // agent.phases.saveDiscovery(answers)     → délègue à PhaseManager
    // agent.phases.proposeFeatures()          → délègue à PhaseManager
    // agent.phases.saveFeatures(validated)    → délègue à PhaseManager
    // agent.phases.generateTasks()            → délègue à PhaseManager
    // agent.phases.validateTask(id, approved) → délègue à PhaseManager
    // agent.phases.setTechStack(stack)        → délègue à PhaseManager
    // agent.versions.list()                   → délègue à VersionManager
    // agent.versions.create(name, desc)       → délègue à VersionManager
    // agent.versions.switch(v)                → délègue à VersionManager
    // agent.versions.complete()               → délègue à VersionManager
    // agent.versions.hotfix(name, reason)     → délègue à VersionManager
    this.phases = {
      saveDiscovery: (answers) => this.phaseManager.saveDiscovery(answers),
      proposeFeatures: () => this.phaseManager.proposeFeatures(),
      saveFeatures: (validated) => this.phaseManager.saveFeatures(validated),
      generateTasks: () => this.phaseManager.generateTasks(),
      validateTask: (id, approved) => this.phaseManager.validateTask(id, approved),
      setTechStack: (stack) => this.phaseManager.setTechStack(stack),
    };

    this.versions = {
      list: () => this.versionManager.list(),
      create: (name, desc) => this.versionManager.create(name, desc),
      switch: (v) => this.versionManager.switch(v),
      complete: (opts) => this.versionManager.complete(opts),
      hotfix: (name, reason) => this.versionManager.hotfix(name, reason),
      status: (v) => this.versionManager.status(v),
    };
  }

  // Initialiser un nouveau projet Workflow
  async init(name) {
    const isAlready = await this.memory.getProject();
    if (isAlready) {
      this.io.warn('Un projet Workflow existe déjà ici. Utilise `workflow start` pour reprendre.');
      return;
    }

    await this.memory.initProject({ name, description: '' });
    this.io.success(`Projet "${name}" initialisé. Lance \`workflow start\` pour commencer.`);
  }

  // Démarrer ou reprendre une session
  async start() {
    await this.checkSync();
    // Intégrer les réponses WatchMode en attente avant de continuer (Phase 3 — tâche 3.5)
    // await WatchMode.processAnswers(this.projectRoot);
    // Afficher le briefing DaemonHeartbeat du jour si disponible (Phase 3 — tâche 3.4)
    // await DaemonHeartbeat.checkBriefing(this.projectRoot, this.io);
    await this.phaseManager.runCurrentPhase();
  }

  // Exécuter la prochaine tâche
  async run() {
    await this.checkSync();

    const project = await this.memory.getProject();
    if (project?.status !== 'ACTIVE') {
      this.io.warn('Les phases projet ne sont pas finalisées. Lance `workflow start` d\'abord.');
      return;
    }

    const techStack = await this.memory.getTechStack();
    const loop = new ExecutionLoop(this.projectRoot, this.llm, techStack);

    // Injecter ExecutionLoop dans ExecutionPhase
    const execPhase = this.phaseManager.phases.ACTIVE;
    execPhase.setExecutionLoop(loop);

    const result = await execPhase.run();

    if (result.versionDone) {
      this.io.success('Version terminée ! Lance `workflow version complete` pour finaliser.');
    }
  }

  // Afficher le statut
  async status() {
    const project = await this.memory.getProject();
    if (!project) {
      this.io.display('Aucun projet Workflow ici. Lance `workflow init "Nom"` pour démarrer.');
      return;
    }

    let versionCtx = null;
    if (project.currentVersion) {
      versionCtx = await this.context.getVersionContext(project.currentVersion);
    }

    this.io.displayStatus(project, versionCtx);
  }

  // Commandes de version (délègue à VersionManager — Phase 5)
  async version(args) {
    const [subcommand, ...rest] = args;
    switch (subcommand) {
      case 'list': return this.versions.list();
      case 'create': return this.versions.create(rest[0], rest.slice(1).join(' '));
      case 'switch': return this.versions.switch(rest[0]);
      case 'complete': return this.versions.complete();
      case 'hotfix': return this.versions.hotfix(rest[0], rest.slice(1).join(' '));
      case 'status': return this.versions.status(rest[0]);
      default:
        this.io.display('Commandes version : list, create, switch, complete, hotfix, status');
    }
  }

  // Onboarding nouveau développeur — résumé complet depuis .workflow/ (v1.5)
  async onboard() {
    // await OnboardingManager.run(this.projectRoot, this.llm, this.io);
    this.io.display('[workflow onboard] disponible en Phase 6 (OnboardingManager)');
  }

  // Mode observation passive — WatchMode + chokidar (tâche 3.5)
  async watch() {
    // Implémenté en tâche 3.5 — WatchMode
    // Stub : afficher un message indiquant que la tâche n'est pas encore complétée
    // await WatchMode.start(this.projectRoot, this.llm);
    this.io.display('workflow watch sera disponible après la tâche 3.5 (WatchMode).');
  }

  // Daemon de surveillance continue (tâche 3.4)
  async daemon() {
    // Implémenté en tâche 3.4 — DaemonHeartbeat
    // await DaemonHeartbeat.start(this.projectRoot, this.llm);
    this.io.display('workflow daemon sera disponible après la tâche 3.4 (DaemonHeartbeat).');
  }

  // Persister les critères validés identifiés par l'analyse LLM du diff
  // Format attendu depuis l'analyse : "TASK-XXX critère N : ..." — une ligne par critère validé
  async _persistManualChanges(analysis, version) {
    if (!version) return;
    const { TaskManager } = await import('../tools/TaskManager.js');
    const tm = new TaskManager(this.projectRoot);

    // Extraire les IDs de tâches mentionnées dans l'analyse
    const taskIds = [...new Set(analysis.match(/TASK-\d{3}/g) ?? [])];

    for (const taskId of taskIds) {
      const task = await tm.getTask(version, taskId);
      if (!task) continue;

      // Vérifier si tous les critères semblent validés → marquer la tâche done
      // Heuristique simple : si l'analyse mentionne le taskId et le critère, on le compte
      const validatedCount = (analysis.match(new RegExp(taskId, 'g')) ?? []).length;
      const totalCriteria = task.criteria?.length ?? 0;

      if (totalCriteria > 0 && validatedCount >= totalCriteria) {
        await tm.markDone(version, taskId);
        this.io.display(`✅ ${taskId} marquée terminée (critères validés depuis le diff).`);
      } else {
        // Ajouter une entrée dans le journal pour traçabilité
        await tm.appendJournal(version, taskId,
          `Modifications manuelles détectées — ${validatedCount}/${totalCriteria} critère(s) validé(s) depuis le diff`
        );
      }
    }
  }

  // Vérification SyncChecker au démarrage
  async checkSync() {
    const report = await this.sync.check();

    switch (report.type) {
      case 'NOT_INITIALIZED':
        this.io.display('Aucun projet Workflow ici. Lance `workflow init "Nom"` pour démarrer.');
        process.exit(0);
        break;

      case 'BRANCH_MISMATCH':
        this.io.warn(report.message);
        const fix = await this.io.confirm('Veux-tu passer à la branche correcte ?');
        if (fix) {
          // Déléguer à VersionManager (disponible depuis Phase 4)
          await this.versionManager.switch(report.activeVersion);
        }
        break;

      case 'MANUAL_CHANGES': {
        this.io.header('Modifications détectées depuis la dernière session');
        this.io.display(`Fichiers modifiés : ${report.files.join(', ')}`);

        // Analyser le diff via LLM pour déterminer quels critères sont validés
        const analysis = await this.llm.ask(
          `Analyse ce diff Git et liste les critères d'acceptation qui semblent être satisfaits.
Réponds en points courts, une ligne par critère validé.
Format : "TASK-XXX critère N : [description du critère]" — un par ligne.

Diff :
${report.diff.substring(0, 3000)}`
        );
        this.io.display('\nAnalyse des changements :');
        this.io.display(analysis);

        const ok = await this.io.confirm('Est-ce correct ?');
        if (ok) {
          // Persister les critères validés dans progress.json via TaskManager
          await this._persistManualChanges(analysis, report.activeVersion);
          // Mettre à jour le timestamp de dernière session pour éviter la re-détection
          await this.memory.updateProject({ lastSessionAt: new Date().toISOString() });
          this.io.display('Progression mise à jour.');
        }
        break;
      }

      case 'DIRTY_REPO':
        this.io.warn(report.message);
        break;

      case 'CLEAN':
        this.io.display(chalk.dim(report.message));
        break;
    }
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `workflow init` bloque si un projet existe déjà | ⬜ |
| 2 | `workflow start` appelle `SyncChecker` en premier | ⬜ |
| 3 | `workflow start` contient les appels commentés WatchMode + DaemonHeartbeat (v1.5) | ⬜ |
| 4 | `workflow run` bloque si les phases ne sont pas `ACTIVE` | ⬜ |
| 5 | `workflow status` affiche projet + version + compteurs de tâches | ⬜ |
| 6 | `workflow onboard`, `workflow watch`, `workflow daemon` existent et affichent un message clair | ⬜ |
| 7 | `MANUAL_CHANGES` soumet le diff au LLM pour analyse | ⬜ |
| 8 | Intégration end-to-end : `init` → `start` → `run` fonctionne sur un projet test | ⬜ |
| 12 | `BRANCH_MISMATCH` appelle `versionManager.switch()` — ne se contente pas d'afficher la commande | ⬜ |
| 13 | `MANUAL_CHANGES` : après confirmation, `progress.json` est mis à jour et `lastSessionAt` actualisé | ⬜ |
| 14 | `_persistManualChanges()` marque une tâche `done` si tous ses critères sont identifiés dans le diff | ⬜ |
| 9 | `WorkflowAgent.phases` expose les 6 méthodes délégant à `PhaseManager` (saveDiscovery, proposeFeatures, saveFeatures, generateTasks, validateTask, setTechStack) | ⬜ |
| 10 | `WorkflowAgent.versions` expose les 5 méthodes délégant à `VersionManager` (list, create, switch, complete, hotfix) | ⬜ |
| 11 | `WorkflowAgent` importe `chalk` — aucune `ReferenceError` sur un repo en état `CLEAN` | ⬜ |
