# Phase 1 — Tâche 1.4 : TaskManager.js

## Objectif

Créer le module `TaskManager.js` qui gère le CRUD sur les fichiers `TASK-XXX.md` et les fichiers `progress.json`. C'est ce module qui impose le format auto-suffisant des tâches et garantit la numérotation séquentielle.

## Dépendances

- Tâche 1.2 ✅ (`FileSystem.js`)
- Tâche 1.3 ✅ (`ProjectMemory.js`)

## Fichiers à Créer / Modifier

- `src/tools/TaskManager.js` [CRÉER]
- `tests/unit/TaskManager.test.js` [CRÉER]

## Format des Fichiers de Tâche

Voir `CLAUDE.md#format-des-tâches-auto-suffisantes` pour le format complet. Le `TaskManager` est responsable de générer et parser ce format.

**Statuts possibles** : `⬜ EN ATTENTE` | `🔄 EN COURS` | `✅ TERMINÉ` | `❌ REPORTÉ`

## Implémentation

```javascript
// src/tools/TaskManager.js
import { FileSystem } from './FileSystem.js';
import { ProjectMemory } from '../core/ProjectMemory.js';

export class TaskManager {
  constructor(projectRoot) {
    this.fs = new FileSystem(projectRoot);
    this.memory = new ProjectMemory(projectRoot);
  }

  // Générer le prochain ID de tâche (TASK-001, TASK-002...)
  async nextTaskId(version) {
    const existing = await this.fs.listTaskIds(version);
    if (existing.length === 0) return 'TASK-001';
    const last = existing[existing.length - 1];
    const num = parseInt(last.replace('TASK-', ''), 10);
    return `TASK-${String(num + 1).padStart(3, '0')}`;
  }

  // Créer une nouvelle tâche
  async createTask(version, taskData) {
    const taskId = taskData.id ?? await this.nextTaskId(version);
    const content = this.renderTaskFile({ ...taskData, id: taskId, version, status: '⬜ EN ATTENTE' });
    await this.fs.writeMarkdown(this.fs.paths.taskFile(version, taskId), content);

    // Ajouter dans progress.json
    const progress = await this.memory.getProgress(version);
    progress.pending.push(taskId);
    await this.memory.saveProgress(version, progress);

    return taskId;
  }

  // Lire une tâche (parser le Markdown → objet)
  async getTask(version, taskId) {
    const content = await this.fs.readMarkdown(this.fs.paths.taskFile(version, taskId));
    if (!content) return null;
    return this.parseTaskFile(taskId, version, content);
  }

  // Marquer une tâche comme terminée
  async markDone(version, taskId) {
    await this.updateTaskStatus(version, taskId, '✅ TERMINÉ');
    const progress = await this.memory.getProgress(version);
    progress.pending = progress.pending.filter(id => id !== taskId);
    progress.done.push(taskId);
    await this.memory.saveProgress(version, progress);
  }

  // Marquer une tâche comme en cours
  async markInProgress(version, taskId) {
    await this.updateTaskStatus(version, taskId, '🔄 EN COURS');
  }

  // Marquer une tâche comme reportée à une autre version
  async deferTask(version, taskId, targetVersion, reason) {
    await this.updateTaskStatus(version, taskId, '❌ REPORTÉ');
    const progress = await this.memory.getProgress(version);
    progress.pending = progress.pending.filter(id => id !== taskId);
    progress.deferred.push({ id: taskId, to: targetVersion, reason });
    await this.memory.saveProgress(version, progress);

    // Ajouter une entrée dans le Journal de la tâche
    await this.appendJournal(version, taskId,
      `Reportée vers ${targetVersion} — raison : ${reason}`
    );
  }

  // Ajouter une entrée dans le champ Journal de la tâche
  async appendJournal(version, taskId, entry) {
    const content = await this.fs.readMarkdown(this.fs.paths.taskFile(version, taskId));
    if (!content) return;
    const date = new Date().toISOString().split('T')[0];
    const updated = content.replace(
      /## Journal\n([\s\S]*?)\n## Statut/,
      `## Journal\n$1[${date}] ${entry}\n\n## Statut`
    );
    await this.fs.writeMarkdown(this.fs.paths.taskFile(version, taskId), updated);
  }

  // Lister les tâches en attente d'une version
  async getPendingTasks(version) {
    const progress = await this.memory.getProgress(version);
    return progress.pending;
  }

  // Prochaine tâche à exécuter (première EN ATTENTE)
  async getNextTask(version) {
    const pending = await this.getPendingTasks(version);
    if (pending.length === 0) return null;
    return this.getTask(version, pending[0]);
  }

  // Mettre à jour le statut dans le fichier Markdown
  async updateTaskStatus(version, taskId, status) {
    const content = await this.fs.readMarkdown(this.fs.paths.taskFile(version, taskId));
    if (!content) return;
    const updated = content.replace(
      /## Statut\n.*/,
      `## Statut\n${status}`
    );
    await this.fs.writeMarkdown(this.fs.paths.taskFile(version, taskId), updated);
  }

  // Rendre un objet tâche en Markdown
  renderTaskFile(task) {
    const deps = (task.dependencies ?? [])
      .map(d => `- ${d.id} ${d.done ? '✅' : '⬜'} (${d.description})`)
      .join('\n') || '(aucune)';

    const files = (task.files ?? [])
      .map(f => `- ${f.path} [${f.action}]`)
      .join('\n') || '(aucun)';

    const criteria = (task.criteria ?? [])
      .map(c => `- [ ] ${c}`)
      .join('\n') || '- [ ] À définir';

    return `# ${task.id} : ${task.title}
## Version : ${task.version}

## Contexte Projet
${task.context ?? '(à compléter)'}

## User Story
${task.userStory ?? '(à compléter)'}

## Dépendances
${deps}

## Fichiers à créer / modifier
${files}

## Critères d'acceptation
${criteria}

## Journal
(vide — tâche jamais tentée)

## Statut
${task.status}
`;
  }

  // Parser un fichier Markdown de tâche → objet
  parseTaskFile(taskId, version, content) {
    const getSection = (name) => {
      const match = content.match(new RegExp(`## ${name}\n([\\s\\S]*?)(?=\n## |$)`));
      return match ? match[1].trim() : '';
    };
    return {
      id: taskId,
      version,
      title: content.match(/^# \S+ : (.+)/m)?.[1] ?? '',
      context: getSection('Contexte Projet'),
      userStory: getSection('User Story'),
      filesToModify: getSection('Fichiers à créer / modifier')
        .split('\n')
        .filter(l => l.startsWith('- '))
        .map(l => l.replace(/^- /, '').split(' [')[0]),
      criteria: getSection("Critères d'acceptation")
        .split('\n')
        .filter(l => l.startsWith('- '))
        .map(l => l.replace(/^- \[.\] /, '')),
      journal: getSection('Journal'),
      status: getSection('Statut'),
    };
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `nextTaskId` génère TASK-001, TASK-002... séquentiellement | ⬜ |
| 2 | `createTask` écrit le fichier Markdown ET met à jour `progress.json` | ⬜ |
| 3 | `markDone` déplace l'ID de `pending` vers `done` | ⬜ |
| 4 | `deferTask` ajoute une entrée dans le champ Journal du fichier | ⬜ |
| 5 | `parseTaskFile` extrait correctement toutes les sections | ⬜ |
| 6 | Tests unitaires couvrent le cycle complet d'une tâche | ⬜ |
