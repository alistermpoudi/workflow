# Phase 3 — Tâche 3.5 : WatchMode.js

## Objectif

Créer le mode d'annotation passive. WatchMode observe les fichiers du projet en arrière-plan et crée des questions dans `.workflow/questions/` quand il détecte des modifications hors du scope de la tâche courante. Zéro interruption — l'utilisateur répond quand il le souhaite.

## Dépendances

- Tâche 3.4 ✅ (DaemonHeartbeat — WatchMode s'intègre dans le daemon)
- Phase 1 ✅

## Fichiers à Créer

- `src/core/WatchMode.js` [CRÉER]
- `tests/unit/WatchMode.test.js` [CRÉER]

## Structure des Questions

```
.workflow/questions/
├── 2026-04-03-Q001.md    # En attente
├── 2026-04-03-Q002.md    # Répondue (oui)
└── 2026-04-03-Q003.md    # Reportée (plus-tard)
```

## Format d'une Question

```markdown
# Q001 — 2026-04-03

**Détection** : Nouveau fichier créé — `src/lib/socket.ts`

Ce fichier référence `socket.io` qui n'est pas dans `tech-stack.json`.

Cela concerne la v1.0 ou la v1.5 ?

**Réponds en remplaçant `[...]` :**
> [v1.0 / v1.5 / ignore / plus-tard]

**Raison (optionnel) :**
>

---
status: pending
detected_at: 2026-04-03T09:14:22Z
file: src/lib/socket.ts
trigger: new_import
import: socket.io
```

## Déclencheurs

| Déclencheur | Condition | Question générée |
|------------|-----------|-----------------|
| `new_file` | Fichier créé dans `src/` hors `filesToModify` de la tâche courante | "Fichier créé sans tâche associée — à quelle version appartient-il ?" |
| `new_import` | Import d'un package non listé dans `tech-stack.json` | "Nouvelle dépendance détectée — je l'ajoute à tech-stack ?" |
| `modified_task_file` | Fichier de tâche `done` modifié manuellement | "Tu as modifié un fichier d'une tâche terminée — je rouvre la tâche ?" |

## Implémentation

```javascript
// src/core/WatchMode.js
import chokidar from 'chokidar';
import { readFile, writeFile, mkdir, readdir } from 'fs/promises';
import { join, relative } from 'path';
import { ProjectMemory } from './ProjectMemory.js';
import { TaskManager } from '../tools/TaskManager.js';

const QUESTIONS_DIR = '.workflow/questions';

export class WatchMode {
  constructor(projectRoot) {
    this.projectRoot = projectRoot;
    this.memory = new ProjectMemory(projectRoot);
    this.tasks = new TaskManager(projectRoot);
    this._questionCounter = 0;
    this._watcher = null;
  }

  async start() {
    // Charger le contexte courant
    const project = await this.memory.getProject();
    const version = project?.currentVersion;
    this._currentTask = version
      ? await this.tasks.getNextTask(version)
      : null;

    // Watcher sur src/ (ignorer .workflow/ et node_modules/)
    this._watcher = chokidar.watch(join(this.projectRoot, 'src'), {
      ignored: [/node_modules/, /\.workflow/],
      persistent: true,
      ignoreInitial: true,
    });

    this._watcher.on('add', (filePath) =>
      this._onFileAdded(filePath).catch(err => console.error('[WatchMode] _onFileAdded error:', err))
    );
    this._watcher.on('change', (filePath) =>
      this._onFileChanged(filePath).catch(err => console.error('[WatchMode] _onFileChanged error:', err))
    );
  }

  stop() {
    this._watcher?.close();
  }

  async _onFileAdded(filePath) {
    const relPath = relative(this.projectRoot, filePath);

    // Vérifier si le fichier est dans le scope de la tâche courante
    if (this._currentTask?.filesToModify?.includes(relPath)) return;

    // Détecter les imports dans le nouveau fichier
    const content = await readFile(filePath, 'utf-8').catch(() => '');
    const newImports = await this._detectNewImports(content);

    if (newImports.length > 0) {
      for (const imp of newImports) {
        await this._createQuestion({
          trigger: 'new_import',
          file: relPath,
          import: imp,
          message: `Ce fichier importe \`${imp}\` qui n'est pas dans \`tech-stack.json\`.\n\nCela concerne la v1.0 ou la v1.5 ?`,
        });
      }
    } else {
      await this._createQuestion({
        trigger: 'new_file',
        file: relPath,
        message: `Fichier créé hors du scope de la tâche courante.\n\nÀ quelle version appartient \`${relPath}\` ?`,
      });
    }
  }

  async _onFileChanged(filePath) {
    const relPath = relative(this.projectRoot, filePath);

    const project = await this.memory.getProject();
    const version = project?.currentVersion;
    if (!version) return;

    // Vérifier si ce fichier appartient à une tâche TERMINÉE (déclencheur modified_task_file)
    const progress = await this.memory.getProgress(version);

    for (const doneTaskId of (progress.done ?? [])) {
      const task = await this.tasks.getTask(version, doneTaskId);
      if (task?.filesToModify?.includes(relPath)) {
        await this._createQuestion({
          trigger: 'modified_task_file',
          file: relPath,
          taskId: doneTaskId,
          message: `\`${relPath}\` appartient à ${doneTaskId} (tâche ✅ terminée).\n\nVeux-tu rouvrir cette tâche pour intégrer cette modification ?`,
        });
        return; // Une question suffit par fichier
      }
    }
  }

  async _createQuestion(data) {
    const date = new Date().toISOString().split('T')[0];
    this._questionCounter++;
    const id = `Q${String(this._questionCounter).padStart(3, '0')}`;
    const filename = `${date}-${id}.md`;

    const triggerLabel = {
      new_file: 'Nouveau fichier',
      new_import: 'Nouvel import',
      modified_task_file: 'Fichier tâche terminée modifié',
    }[data.trigger] ?? data.trigger;

    // Pour modified_task_file : réponse oui/non au lieu de v1.0/v1.5
    const responseFormat = data.trigger === 'modified_task_file'
      ? '> [oui / non]'
      : '> [v1.0 / v1.5 / ignore / plus-tard]';

    const content = `# ${id} — ${date}\n\n**Détection** : ${triggerLabel} — \`${data.file}\`\n\n${data.message}\n\n**Réponds en remplaçant \`[...]\` :**\n${responseFormat}\n\n**Raison (optionnel) :**\n>\n\n---\nstatus: pending\ndetected_at: ${new Date().toISOString()}\nfile: ${data.file}\ntrigger: ${data.trigger}\n${data.import ? `import: ${data.import}` : ''}${data.taskId ? `task_id: ${data.taskId}` : ''}\n`;

    await mkdir(join(this.projectRoot, QUESTIONS_DIR), { recursive: true });
    await writeFile(join(this.projectRoot, QUESTIONS_DIR, filename), content, 'utf-8');
  }

  async _detectNewImports(content) {
    // Lire les dépendances connues depuis package.json (pas tech-stack.json qui n'a pas de champ 'dependencies')
    let knownDeps = [];
    try {
      const pkgRaw = await readFile(join(this.projectRoot, 'package.json'), 'utf-8');
      const pkg = JSON.parse(pkgRaw);
      knownDeps = Object.keys({ ...(pkg.dependencies ?? {}), ...(pkg.devDependencies ?? {}) });
    } catch {
      // package.json absent ou invalide — pas de filtrage possible
    }

    const importRegex = /(?:import|require)\s+(?:.*?\s+from\s+)?['"]([^./][^'"]+)['"]/g;
    const found = new Set();
    let match;
    while ((match = importRegex.exec(content)) !== null) {
      const pkg = match[1].split('/')[0]; // @scope/pkg → @scope/pkg
      if (!knownDeps.includes(pkg)) found.add(pkg);
    }
    return [...found];
  }

  // Intégrer les réponses aux questions au démarrage
  // Prend uniquement projectRoot — instancie ses dépendances en interne
  static async processAnswers(projectRoot) {
    const { ProjectMemory } = await import('./ProjectMemory.js');
    const { TaskManager } = await import('../tools/TaskManager.js');
    const { DecisionsLog } = await import('./DecisionsLog.js');

    const tasks = new TaskManager(projectRoot);
    const decisions = new DecisionsLog(projectRoot);

    const questionsDir = join(projectRoot, QUESTIONS_DIR);
    let files;
    try { files = await readdir(questionsDir); }
    catch { return; } // Dossier absent = pas de questions

    const pending = files.filter(f => f.endsWith('.md'));
    for (const file of pending) {
      const content = await readFile(join(questionsDir, file), 'utf-8');
      if (!content.includes('status: pending')) continue;

      // Format de réponse : > [v1.0 / v1.5 / ignore / plus-tard] ou > [oui / non] pour modified_task_file
      const answerMatch = content.match(/>\s*\[(v\d+\.\d+|ignore|plus-tard|oui|non)\]/);
      if (!answerMatch) continue; // Pas encore répondu → skip

      const answer = answerMatch[1];
      await WatchMode._applyAnswer(file, answer, content, questionsDir, tasks, decisions);
    }
  }

  // Appliquer une réponse à une question WatchMode
  // answer : 'v1.0' | 'v1.5' | 'ignore' | 'plus-tard' | 'oui' | 'non'
  static async _applyAnswer(file, answer, content, questionsDir, tasks, decisions) {
    const filePath = join(questionsDir, file);

    if (answer === 'plus-tard') return; // Garder pour la prochaine session

    if (answer === 'ignore' || answer === 'non') {
      // Marquer la question comme résolue sans action
      const updated = content.replace('status: pending', 'status: ignored');
      await writeFile(filePath, updated, 'utf-8');
      return;
    }

    // Réponse 'oui' — pour les questions modified_task_file : rouvrir la tâche
    if (answer === 'oui') {
      const taskIdMatch = content.match(/^task_id: (.+)$/m);
      const fileMatch = content.match(/^file: (.+)$/m);
      const taskId = taskIdMatch?.[1]?.trim();
      const relPath = fileMatch?.[1]?.trim();

      if (taskId && decisions) {
        await decisions.log(
          'WATCH',
          `Tâche ${taskId} rouverte`,
          `Fichier ${relPath} modifié après completion — l'utilisateur a confirmé la réouverture`
        );
      }
      // Note : TaskManager.reopenTask() sera implémenté si nécessaire
      // Pour l'instant : enregistrer la décision suffit, l'agent prendra en charge au prochain workflow run
      const updated = content.replace('status: pending', `status: resolved\naction: reopen_task`);
      await writeFile(filePath, updated, 'utf-8');
      return;
    }

    // Réponse = version (v1.0, v1.5, etc.) → créer une tâche dans cette version
    const version = answer;
    const fileMatch = content.match(/^file: (.+)$/m);
    const triggerMatch = content.match(/^trigger: (.+)$/m);
    const importMatch = content.match(/^import: (.+)$/m);

    const relPath = fileMatch?.[1]?.trim() ?? 'unknown';
    const trigger = triggerMatch?.[1]?.trim() ?? 'new_file';
    const importName = importMatch?.[1]?.trim();

    const taskData = {
      title: trigger === 'new_import'
        ? `Intégrer la dépendance \`${importName}\` dans tech-stack`
        : `Intégrer le fichier \`${relPath}\` dans la version ${version}`,
      context: 'Fichier détecté hors scope de la tâche courante par WatchMode',
      userStory: `EN TANT QUE développeur\nJE VEUX que ce fichier soit tracé dans Workflow\nAFIN DE maintenir la cohérence du projet`,
      intent: `Fichier ${relPath} créé manuellement. L'utilisateur confirme qu'il appartient à la version ${version}.`,
      files: [{ path: relPath, action: 'MODIFIER' }],
      criteria: [
        'Le fichier est documenté dans une tâche Workflow',
        `La tâche est assignée à la version ${version}`,
      ],
    };

    await tasks.createTask(version, taskData);

    // Si nouvel import → enregistrer la décision dans decisions.log
    if (importName) {
      await decisions.log('WATCH', `Nouvelle dépendance : ${importName}`, `Détectée dans ${relPath}, confirmée pour la version ${version}`);
    }

    // Marquer la question comme résolue
    const updated = content.replace(
      'status: pending',
      `status: resolved\nresolved_version: ${version}`
    );
    await writeFile(filePath, updated, 'utf-8');
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | Un fichier créé hors-tâche génère une question dans `.workflow/questions/` | ⬜ |
| 2 | Un import inconnu déclenche une question spécifique à cet import | ⬜ |
| 3 | Les fichiers dans `filesToModify` de la tâche courante n'ont pas de question | ⬜ |
| 4 | `workflow start` appelle `WatchMode.processAnswers(projectRoot)` avant de continuer | ⬜ |
| 5 | Réponse `plus-tard` conserve la question (`status: pending`) pour la prochaine session | ⬜ |
| 6 | Réponse `v1.5` crée une nouvelle tâche dans la version correspondante | ⬜ |
| 7 | Réponse `ignore` écrit `status: ignored` dans le fichier question | ⬜ |
| 8 | `_applyAnswer()` enregistre une décision dans `decisions.log` si trigger = `new_import` | ⬜ |
| 9 | `processAnswers()` skip silencieusement si `.workflow/questions/` n'existe pas | ⬜ |
| 10 | `_onFileChanged()` crée une question si le fichier appartient à une tâche `done` | ⬜ |
| 11 | `_detectNewImports()` lit `package.json#dependencies` — n'utilise pas `tech-stack.json` | ⬜ |
| 12 | `_createQuestion()` utilise le format `[oui / non]` pour `modified_task_file` (pas `[v1.0 / v1.5]`) | ⬜ |
| 15 | `processAnswers()` matche les réponses `oui` et `non` (regex inclut `oui\|non`) | ⬜ |
| 16 | `_applyAnswer('non', ...)` marque `status: ignored` sans créer de tâche | ⬜ |
| 17 | `_applyAnswer('oui', ...)` enregistre une décision et marque `action: reopen_task` | ⬜ |
| 13 | Les handlers chokidar (`add`, `change`) catchent les erreurs async — pas d'UnhandledPromiseRejection | ⬜ |
| 14 | Le watcher ignore `node_modules/` et `.workflow/` | ⬜ |
| 15 | Tests unitaires mockent chokidar | ⬜ |
