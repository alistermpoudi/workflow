# Phase 3 — Tâche 3.4 : DaemonHeartbeat.js

## Objectif

Créer le daemon Workflow qui tourne en arrière-plan en permanence. C'est lui qui transforme Workflow d'un outil réactif (on l'appelle) en outil proactif (il prévient). Le daemon surveille l'état du projet et prépare un briefing au démarrage de journée.

## Dépendances

- Tâche 3.1 ✅ (ExecutionLoop)
- Tâche 3.2 ✅ (CLI)
- Tâche 3.3 ✅ (WorkflowAgent)

## Fichiers à Créer

- `src/core/DaemonHeartbeat.js` [CRÉER]
- `src/interfaces/DaemonCLI.js` [CRÉER] — commandes daemon start/stop/status
- `tests/unit/DaemonHeartbeat.test.js` [CRÉER]

## Responsabilités

- Tourner en arrière-plan (launchd sur macOS, systemd sur Linux, Task Scheduler sur Windows)
- Vérifier l'état du projet toutes les X minutes (configurable)
- Générer un briefing quotidien au premier démarrage de journée
- Détecter les builds CI qui cassent (si GitHub Integration disponible)
- Proposer la prochaine tâche quand une est terminée
- Ne jamais bloquer — notifications uniquement, pas d'actions automatiques

## Implémentation

```javascript
// src/core/DaemonHeartbeat.js
import { ProjectMemory } from './ProjectMemory.js';
import { TaskManager } from '../tools/TaskManager.js';
import { DecisionsLog } from './DecisionsLog.js';
import { readFile, writeFile, mkdir } from 'fs/promises';
import { join } from 'path';

const BRIEFING_DIR = '.workflow/briefings';
const CHECK_INTERVAL_MS = 5 * 60 * 1000; // 5 minutes

export class DaemonHeartbeat {
  constructor(projectRoot) {
    this.projectRoot = projectRoot;
    this.memory = new ProjectMemory(projectRoot);
    this.tasks = new TaskManager(projectRoot);
    this.decisions = new DecisionsLog(projectRoot);
    this._timer = null;
    this._lastBriefingDate = null;
    this._lastDoneCount = undefined; // Undefined = pas encore observé
  }

  // Démarrer le daemon
  start() {
    this._tick(); // Premier tick immédiat
    this._timer = setInterval(() => this._tick(), CHECK_INTERVAL_MS);
    process.on('SIGTERM', () => this.stop());
    process.on('SIGINT', () => this.stop());
  }

  stop() {
    if (this._timer) clearInterval(this._timer);
    process.exit(0);
  }

  async _tick() {
    try {
      const project = await this.memory.getProject();
      if (!project) return; // Pas de projet initialisé

      // Générer le briefing une fois par jour (premier tick du matin)
      const today = new Date().toISOString().split('T')[0];
      if (this._lastBriefingDate !== today && this._isMorning()) {
        await this._generateBriefing(project);
        this._lastBriefingDate = today;
      }

      // Vérifier les tâches terminées → notifier la suivante
      await this._checkTaskCompletion(project);

    } catch (err) {
      // Le daemon ne doit jamais crasher — log silencieux
      await this._logError(err);
    }
  }

  _isMorning() {
    const hour = new Date().getHours();
    return hour >= 7 && hour <= 10;
  }

  async _generateBriefing(project) {
    const version = project.currentVersion;
    if (!version) return;

    const progress = await this.memory.getProgress(version);
    const total = progress.done.length + progress.pending.length;
    if (total === 0) return; // Pas de tâches → division par zéro dans le calcul de pourcentage
    const nextTask = progress.pending[0]
      ? await this.tasks.getTask(version, progress.pending[0])
      : null;

    const recentDecisions = await this.decisions.getRecent(3);

    const briefing = this._renderBriefing({
      project,
      version,
      done: progress.done.length,
      total: progress.done.length + progress.pending.length,
      nextTask,
      recentDecisions,
      date: new Date().toISOString().split('T')[0],
    });

    // Écrire le briefing dans .workflow/briefings/YYYY-MM-DD.md
    await mkdir(join(this.projectRoot, BRIEFING_DIR), { recursive: true });
    const path = join(this.projectRoot, BRIEFING_DIR, `${new Date().toISOString().split('T')[0]}.md`);
    await writeFile(path, briefing, 'utf-8');
  }

  _renderBriefing({ project, version, done, total, nextTask, recentDecisions, date }) {
    const pct = Math.round((done / total) * 100);
    const bar = '█'.repeat(Math.round(pct / 10)) + '░'.repeat(10 - Math.round(pct / 10));

    let content = `# Workflow — Briefing ${date}\n\n`;
    content += `**Projet** : ${project.name}\n`;
    content += `**Version** : ${version} ACTIVE\n`;
    content += `**Avancement** : ${done}/${total} tâches ${bar} ${pct}%\n\n`;

    if (nextTask) {
      content += `## Prochaine Tâche\n\n`;
      content += `**${nextTask.id}** : ${nextTask.title}\n`;
      if (nextTask.intent) content += `\n> ${nextTask.intent}\n`;
    }

    if (recentDecisions.length > 0) {
      content += `\n## Décisions Récentes\n\n`;
      recentDecisions.forEach(d => content += `- ${d.summary}\n`);
    }

    return content;
  }

  async _checkTaskCompletion(project) {
    const version = project.currentVersion;
    if (!version) return;

    const progress = await this.memory.getProgress(version);
    const currentDoneCount = progress.done.length;

    // Premier tick — initialiser le compteur sans notifier
    if (this._lastDoneCount === undefined) {
      this._lastDoneCount = currentDoneCount;
      return;
    }

    if (currentDoneCount > this._lastDoneCount) {
      // Une ou plusieurs tâches viennent d'être marquées done
      const lastDoneId = progress.done[progress.done.length - 1];
      const nextTaskId = progress.pending[0];

      const content = `**${lastDoneId}** terminée !\n\n` +
        (nextTaskId
          ? `Prochaine tâche : **${nextTaskId}**\nLance \`workflow run\` pour continuer.`
          : `Toutes les tâches sont terminées. Lance \`workflow version complete\`.`
        );

      // Écrire notification dans .workflow/notifications/
      const notifDir = join(this.projectRoot, '.workflow', 'notifications');
      await mkdir(notifDir, { recursive: true });
      const ts = new Date().toISOString().replace(/[:.]/g, '-');
      await writeFile(join(notifDir, `${ts}.md`), content, 'utf-8');
    }

    this._lastDoneCount = currentDoneCount;
  }

  async _logError(err) {
    const logPath = join(this.projectRoot, '.workflow', 'daemon.log');
    const entry = `[${new Date().toISOString()}] ERROR: ${err.message}\n`;
    await writeFile(logPath, entry, { flag: 'a' });
  }

  // Méthode statique — appelée au démarrage d'une session (WorkflowAgent.start, PhaseManager.runCurrentPhase)
  // Lit le briefing du jour dans .workflow/briefings/YYYY-MM-DD.md et l'affiche si présent
  static async checkBriefing(projectRoot, io) {
    const today = new Date().toISOString().split('T')[0];
    const briefingPath = join(projectRoot, BRIEFING_DIR, `${today}.md`);
    try {
      const content = await readFile(briefingPath, 'utf-8');
      io.header(`Briefing du ${today}`);
      io.display(content);
    } catch {
      // Pas de briefing pour aujourd'hui — silencieux (daemon pas encore démarré ou hors plage 7h-10h)
    }
  }
}
```

## Commandes CLI

```bash
workflow daemon start    # Démarre le daemon en arrière-plan
workflow daemon stop     # Arrête le daemon
workflow daemon status   # Affiche si le daemon tourne + dernière activité
workflow briefing        # Affiche le briefing du jour (sans daemon)
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `workflow daemon start` persiste après fermeture du terminal | ⬜ |
| 2 | Le briefing est généré une seule fois par jour (7h-10h) | ⬜ |
| 3 | Le briefing contient : projet, version, avancement, prochaine tâche | ⬜ |
| 4 | Le daemon ne crashe pas si `.workflow/` est absent | ⬜ |
| 5 | `workflow daemon stop` arrête proprement le process | ⬜ |
| 6 | `workflow daemon status` indique running/stopped + dernière activité | ⬜ |
| 7 | Les erreurs internes sont loggées dans `.workflow/daemon.log` sans crash | ⬜ |
| 8 | Tests unitaires mockent le timer et vérifient la logique de briefing | ⬜ |
| 9 | `DaemonHeartbeat.checkBriefing(projectRoot, io)` affiche le briefing du jour s'il existe | ⬜ |
| 10 | `DaemonHeartbeat.checkBriefing()` ne throw pas si `.workflow/briefings/` est absent | ⬜ |
| 11 | `_generateBriefing()` retourne sans crash si `done=0` et `pending=0` (total=0) | ⬜ |
