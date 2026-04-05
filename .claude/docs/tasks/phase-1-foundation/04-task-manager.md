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

### Champ `Intent` — Le "Pourquoi Humain"

Chaque tâche inclut une section `## Intent` distincte des critères d'acceptation. Elle capture **pourquoi l'utilisateur veut vraiment cette fonctionnalité** — informations qui guident les décisions d'implémentation quand les critères sont ambigus.

```markdown
## Intent
L'utilisateur veut que les clients puissent s'inscrire seuls,
SANS passer par un admin. Le flux email de confirmation est
secondaire pour l'instant — ne pas bloquer l'inscription
sur une vérification email non reçue.

## Critères d'acceptation
- [ ] POST /auth/register fonctionne
- [ ] Email de confirmation envoyé (non-bloquant)
```

L'`intent` est injecté dans le prompt LLM **avant** les critères. Si deux implémentations respectent les critères mais l'une contredit l'intent, Workflow choisit l'autre.

**Règle** : L'`intent` ne peut pas être vide sur une tâche ayant des critères d'acceptation définis. `ValidationPhase` l'exige lors de la génération.

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

    // Remplacer "(vide — tâche jamais tentée)" par la première entrée, ou appender
    let updated;
    if (content.includes('(vide — tâche jamais tentée)')) {
      updated = content.replace(
        '(vide — tâche jamais tentée)',
        `[${date}] ${entry}`
      );
    } else {
      // Appender avant ## Statut (avec ou sans ligne vide avant)
      updated = content.replace(
        /(\n{1,2}## Statut)/,
        `\n[${date}] ${entry}\n$1`
      );
    }

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

    const preconditions = task.preconditions
      ? Object.entries(task.preconditions)
          .map(([k, v]) => `- ${k}: ${JSON.stringify(v)}`)
          .join('\n')
      : '(aucune)';

    // Rendu du mockup UI — présent uniquement pour les tâches avec interface
    const mockupSection = task.mockup?.screens?.length
      ? task.mockup.screens
          .map(s => `### Écran — ${s.name}\n${s.ascii}\n${s.notes ? `Style : ${s.notes}` : ''}`)
          .join('\n\n')
      : '(aucune interface — tâche backend / configuration)';

    return `# ${task.id} : ${task.title}
## Version : ${task.version}

## Contexte Projet
${task.context ?? '(à compléter)'}

## User Story
${task.userStory ?? '(à compléter)'}

## Intent
${task.intent ?? '(à compléter — pourquoi l\'utilisateur veut vraiment cette fonctionnalité)'}

## Préconditions
${preconditions}

## Dépendances
${deps}

## Fichiers à créer / modifier
${files}

## Critères d'acceptation
${criteria}

## Mockup UI
${mockupSection}

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
    // Parser les préconditions (format "- clé: valeur")
    const preconditionsRaw = getSection('Préconditions');
    const preconditions = {};
    if (preconditionsRaw && preconditionsRaw !== '(aucune)') {
      for (const line of preconditionsRaw.split('\n').filter(l => l.startsWith('- '))) {
        const [key, ...rest] = line.replace(/^- /, '').split(':');
        try {
          preconditions[key.trim()] = JSON.parse(rest.join(':').trim());
        } catch {
          preconditions[key.trim()] = rest.join(':').trim();
        }
      }
    }

    // Parser les mockups (format "### Écran — NomÉcran\n[ascii]\nStyle : notes")
    const mockupRaw = getSection('Mockup UI');
    let mockup = null;
    if (mockupRaw && !mockupRaw.startsWith('(aucune')) {
      const screens = [];
      const screenBlocks = mockupRaw.split(/\n(?=### Écran — )/);
      for (const block of screenBlocks) {
        const nameMatch = block.match(/^### Écran — (.+)/);
        if (!nameMatch) continue;
        const name = nameMatch[1].trim();
        const rest = block.slice(block.indexOf('\n') + 1);
        const styleMatch = rest.match(/\nStyle : (.+)$/m);
        const notes = styleMatch?.[1]?.trim() ?? null;
        const ascii = styleMatch
          ? rest.slice(0, rest.lastIndexOf('\nStyle :')).trim()
          : rest.trim();
        screens.push({ name, ascii, notes });
      }
      if (screens.length > 0) mockup = { screens };
    }

    return {
      id: taskId,
      version,
      title: content.match(/^# \S+ : (.+)/m)?.[1] ?? '',
      context: getSection('Contexte Projet'),
      userStory: getSection('User Story'),
      intent: getSection('Intent'),
      preconditions: Object.keys(preconditions).length > 0 ? preconditions : null,
      filesToModify: getSection('Fichiers à créer / modifier')
        .split('\n')
        .filter(l => l.startsWith('- '))
        .map(l => l.replace(/^- /, '').split(' [')[0]),
      criteria: getSection("Critères d'acceptation")
        .split('\n')
        .filter(l => l.startsWith('- '))
        .map(l => l.replace(/^- \[.\] /, '')),
      mockup,
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
| 4b | `appendJournal` remplace correctement `(vide — tâche jamais tentée)` à la première entrée | ⬜ |
| 4c | `appendJournal` appende correctement une 2ème entrée sous la première | ⬜ |
| 4d | `appendJournal` fonctionne que `## Statut` soit précédé d'une ou deux lignes vides | ⬜ |
| 5 | `parseTaskFile` extrait correctement toutes les sections dont `intent` et `preconditions` | ⬜ |
| 6 | `renderTaskFile` inclut les sections `## Intent` ET `## Préconditions` dans le Markdown | ⬜ |
| 7 | `parseTaskFile` retourne `preconditions: null` si la section est absente ou vide | ⬜ |
| 8 | Tests unitaires couvrent le cycle complet d'une tâche | ⬜ |
| 9 | `renderTaskFile` inclut la section `## Mockup UI` dans tous les fichiers générés | ⬜ |
| 10 | Si `task.mockup` est null ou vide, la section affiche `(aucune interface — tâche backend / configuration)` | ⬜ |
| 11 | `parseTaskFile` extrait correctement `mockup.screens[].name`, `.ascii`, `.notes` | ⬜ |
| 12 | `parseTaskFile` retourne `mockup: null` si la section mockup commence par `(aucune` | ⬜ |
