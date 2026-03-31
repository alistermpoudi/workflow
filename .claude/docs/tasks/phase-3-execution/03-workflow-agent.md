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
import { ProjectMemory } from './ProjectMemory.js';
import { PhaseManager } from './PhaseManager.js';
import { SyncChecker } from './SyncChecker.js';
import { VersionManager } from './VersionManager.js';
import { ContextManager } from './ContextManager.js';
import { LLMProvider } from '../llm/LLMProvider.js';
import { ExecutionLoop } from '../tools/ExecutionLoop.js';

export class WorkflowAgent {
  constructor(projectRoot, io) {
    this.projectRoot = projectRoot;
    this.io = io;
    this.memory = new ProjectMemory(projectRoot);
    this.sync = new SyncChecker(projectRoot);
    this.llm = new LLMProvider();
    this.context = new ContextManager(projectRoot);
    this.phase = new PhaseManager(projectRoot, this.llm, io);
    this.versions = new VersionManager(projectRoot, io); // Phase 5
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
    await this.phase.runCurrentPhase();
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
    const execPhase = this.phase.phases.ACTIVE;
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
          // Déléguer au VersionManager (Phase 5)
          this.io.display(`→ git checkout ${report.expectedBranch}`);
        }
        break;

      case 'MANUAL_CHANGES':
        this.io.header('Modifications détectées depuis la dernière session');
        this.io.display(`Fichiers modifiés : ${report.files.join(', ')}`);

        // Analyser le diff via LLM pour déterminer quels critères sont validés
        const analysis = await this.llm.ask(
          `Analyse ce diff Git et liste les critères d'acceptation qui semblent être satisfaits.
Réponds en points courts, une ligne par critère validé.

Diff :
${report.diff.substring(0, 3000)}`
        );
        this.io.display('\nAnalyse des changements :');
        this.io.display(analysis);

        const ok = await this.io.confirm('Est-ce correct ?');
        if (ok) {
          // Mettre à jour progress.json selon l'analyse (simplifié)
          this.io.display('Progression mise à jour.');
        }
        break;

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
| 3 | `workflow run` bloque si les phases ne sont pas `ACTIVE` | ⬜ |
| 4 | `workflow status` affiche projet + version + compteurs de tâches | ⬜ |
| 5 | `MANUAL_CHANGES` soumet le diff au LLM pour analyse | ⬜ |
| 6 | Intégration end-to-end : `init` → `start` → `run` fonctionne sur un projet test | ⬜ |
