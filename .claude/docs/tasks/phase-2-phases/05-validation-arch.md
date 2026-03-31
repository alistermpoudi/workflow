# Phase 2 — Tâche 2.5 : ValidationPhase + ArchitecturePhase

## Objectif

`ValidationPhase` génère les fichiers de tâches version par version et les soumet à validation. `ArchitecturePhase` définit la stack technique et impose TASK-001 + TASK-002.

## Dépendances

- Tâches 2.1 à 2.4 ✅

## Fichiers à Créer

- `src/phases/ValidationPhase.js` [CRÉER]
- `src/phases/ArchitecturePhase.js` [CRÉER]

## `ValidationPhase.js`

```javascript
// src/phases/ValidationPhase.js
import { ProjectMemory } from '../core/ProjectMemory.js';
import { TaskManager } from '../tools/TaskManager.js';
import { PromptBuilder } from '../llm/PromptBuilder.js';

const GRANULARITY_RULE = `
RÈGLE DE GRANULARITÉ OBLIGATOIRE :
1 tâche = 1 PR potentielle = max 4 heures de travail = max 3 fichiers créés ou modifiés.
Si une fonctionnalité dépasse ces seuils, découpe-la en sous-tâches numérotées.
Justifie chaque découpe.
`;

export class ValidationPhase {
  constructor(projectRoot, llm, io) {
    this.memory = new ProjectMemory(projectRoot);
    this.tasks = new TaskManager(projectRoot);
    this.llm = llm;
    this.io = io;
  }

  async run() {
    const features = await this.memory.getFeatures();
    const techStack = await this.memory.getTechStack();
    if (!features) throw new Error('features.json manquant — relancer SpecificationPhase');
    if (!techStack) throw new Error('tech-stack.json manquant — relancer ArchitecturePhase');

    const versions = Object.keys(features);

    for (const version of versions) {
      await this.generateAndValidateVersion(version, features, techStack);
    }

    return { completed: true };
  }

  async generateAndValidateVersion(version, features, techStack) {
    this.io.display(`\n--- Génération des tâches pour ${version} ---`);

    const existingIds = await this.tasks.getPendingTasks(version);

    // Générer les tâches via LLM
    const prompt = PromptBuilder.generateTasks(version, features, techStack, existingIds)
      + '\n\n' + GRANULARITY_RULE;

    const tasksJson = await this.llm.ask(prompt);
    let tasksList;
    try {
      tasksList = JSON.parse(tasksJson);
    } catch {
      throw new Error('LLM a retourné un JSON invalide pour les tâches');
    }

    // Afficher le résumé
    this.io.display(`\n${version} — ${tasksList.length} tâches générées :`);
    tasksList.forEach((t, i) => {
      const fileCount = t.files?.length ?? 0;
      const flag = fileCount > 3 ? ' ⚠️ >3 fichiers' : '';
      this.io.display(`  ${t.id} : ${t.title} (${fileCount} fichiers)${flag}`);
    });

    const approved = await this.io.confirm(`\nCes tâches pour ${version} te conviennent ? (o/n)`);

    if (!approved) {
      const corrections = await this.io.ask('Quelles modifications ? (ex: découpe TASK-003, fusionne TASK-007 et TASK-008)');
      // Régénérer avec corrections
      const correctedPrompt = prompt + `\n\nModifications demandées : ${corrections}`;
      const corrected = JSON.parse(await this.llm.ask(correctedPrompt));
      tasksList = corrected;
    }

    // Sauvegarder les tâches validées
    for (const taskData of tasksList) {
      await this.tasks.createTask(version, taskData);
    }

    // Créer meta.json pour la version
    await this.memory.saveVersionMeta(version, {
      title: version,
      description: `Fonctionnalités ${version}`,
      status: 'DRAFT',
      branch: `workflow/${version}`,
      createdAt: new Date().toISOString(),
    });

    this.io.display(`✅ ${tasksList.length} tâches créées pour ${version}.`);
  }
}
```

## `ArchitecturePhase.js`

```javascript
// src/phases/ArchitecturePhase.js
import { ProjectMemory } from '../core/ProjectMemory.js';
import { TaskManager } from '../tools/TaskManager.js';

export class ArchitecturePhase {
  constructor(projectRoot, llm, io) {
    this.memory = new ProjectMemory(projectRoot);
    this.tasks = new TaskManager(projectRoot);
    this.llm = llm;
    this.io = io;
  }

  async run() {
    const vision = await this.memory.getVision();

    // 1. Proposer une stack si non définie
    const stackPrompt = `Sur la base de cette vision, propose la stack technologique optimale.
Retourne un JSON : { "language": "...", "framework": "...", "database": "...", "runtime": "...",
"build_validate": "commande de build+lint", "test": "commande de test",
"allowed_commands": ["cmd1", "cmd2", ...] }

Vision : ${vision}

Justifie brièvement chaque choix. Retourne JSON + justifications séparées.`;

    const response = await this.llm.ask(stackPrompt);
    // Parser la partie JSON de la réponse
    const jsonMatch = response.match(/```json\n([\s\S]+?)\n```/) ||
                      response.match(/\{[\s\S]+\}/);
    const techStack = JSON.parse(jsonMatch?.[1] ?? jsonMatch?.[0] ?? response);

    // 2. Afficher pour validation
    this.io.display('\n--- STACK PROPOSÉE ---');
    this.io.display(`Langage    : ${techStack.language}`);
    this.io.display(`Framework  : ${techStack.framework}`);
    this.io.display(`Base de données : ${techStack.database}`);
    this.io.display(`Build/Validate  : ${techStack.build_validate}`);
    this.io.display(`Tests      : ${techStack.test}`);
    this.io.display(`Commandes autorisées : ${techStack.allowed_commands?.join(', ')}`);

    const approved = await this.io.confirm('\nCette stack te convient ? (o/n)');

    if (!approved) {
      const corrections = await this.io.ask('Quelles modifications ? (ex: utilise Bun au lieu de Node, PostgreSQL au lieu de SQLite)');
      // Re-générer avec corrections (simplifié)
      const corrected = JSON.parse(await this.llm.ask(stackPrompt + `\nModifications : ${corrections}. Retourne UNIQUEMENT le JSON.`));
      Object.assign(techStack, corrected);
    }

    // 3. Sauvegarder tech-stack.json
    await this.memory.saveTechStack(techStack);

    // 4. Imposer TASK-001 et TASK-002 en première position de la v1.0
    await this.injectFoundationTasks(techStack);

    this.io.display('✅ tech-stack.json sauvegardé.');
    this.io.display('✅ TASK-001 (setup + linter) et TASK-002 (tests + smoke) ajoutés en tête de v1.0.');

    return { completed: true };
  }

  // Injecter TASK-001 et TASK-002 systématiquement en tête de v1.0
  async injectFoundationTasks(techStack) {
    const versions = await this.memory.listVersions();
    const v1 = versions.find(v => v === 'v1.0') ?? 'v1.0';

    // Vérifier que ces tâches n'existent pas déjà
    const existing = await this.tasks.getPendingTasks(v1);
    if (existing.includes('TASK-001')) return;

    await this.tasks.createTask(v1, {
      id: 'TASK-001',
      title: 'Setup du projet et configuration du linter',
      context: `Application: ${(await this.memory.getProject())?.name}\nStack: ${techStack.language}/${techStack.framework}`,
      userStory: `EN TANT QUE développeur\nJE VEUX un projet initialisé avec linter configuré\nAFIN DE garantir la qualité du code dès le départ`,
      dependencies: [],
      files: [
        { path: 'package.json', action: 'CRÉER' },
        { path: `${techStack.lintConfig ?? 'eslint.config.js'}`, action: 'CRÉER' },
      ],
      criteria: [
        `${techStack.build_validate} passe sans erreur`,
        'Structure de dossiers conforme à la stack',
        'README.md avec les instructions de démarrage',
      ],
    });

    await this.tasks.createTask(v1, {
      id: 'TASK-002',
      title: 'Configuration du framework de tests avec smoke test',
      context: `Application: ${(await this.memory.getProject())?.name}\nStack: ${techStack.language}/${techStack.framework}`,
      userStory: `EN TANT QUE développeur\nJE VEUX un framework de tests configuré avec un premier test\nAFIN DE valider que l'infra de test fonctionne`,
      dependencies: [{ id: 'TASK-001', done: false, description: 'Setup projet' }],
      files: [
        { path: 'tests/smoke.test.js', action: 'CRÉER' },
        { path: 'vitest.config.js', action: 'CRÉER' },
      ],
      criteria: [
        `${techStack.test} passe avec au moins 1 test`,
        'Le test de smoke vérifie que le projet démarre sans erreur',
      ],
    });
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `ValidationPhase` applique la règle de granularité dans le prompt | ⬜ |
| 2 | `ValidationPhase` affiche un avertissement si une tâche a >3 fichiers | ⬜ |
| 3 | `ArchitecturePhase` sauvegarde `tech-stack.json` avec `allowed_commands` | ⬜ |
| 4 | TASK-001 et TASK-002 sont toujours créées en tête de v1.0 | ⬜ |
| 5 | TASK-001/002 ne sont pas créées en double si elles existent déjà | ⬜ |
