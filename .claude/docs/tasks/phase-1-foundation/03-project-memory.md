# Phase 1 — Tâche 1.3 : ProjectMemory.js

## Objectif

Créer le module `ProjectMemory.js` — la couche de lecture/écriture de tous les fichiers de métadonnées du projet (sauf les tâches, gérées par `TaskManager`). C'est le module que tous les autres consultent pour connaître l'état du projet.

## Dépendances

- Tâche 1.2 ✅ (`FileSystem.js`)

## Fichiers à Créer / Modifier

- `src/core/ProjectMemory.js` [CRÉER]
- `tests/unit/ProjectMemory.test.js` [CRÉER]

## Responsabilités

- Créer un nouveau projet (initialiser tous les fichiers `.workflow/`)
- Lire/écrire `project.json`, `vision.md`, `features.json`, `tech-stack.json`
- Lire/écrire les `meta.json` et `progress.json` de chaque version
- Retourner un résumé court du projet (pour `ContextManager` — ~500 tokens max)
- Gérer la version active

## Implémentation

```javascript
// src/core/ProjectMemory.js
import { FileSystem } from '../tools/FileSystem.js';

export class ProjectMemory {
  constructor(projectRoot) {
    this.fs = new FileSystem(projectRoot);
  }

  // Initialiser un nouveau projet
  async initProject(data) {
    // fs.init() crée .workflow/, versions/, questions/ et briefings/
    await this.fs.init();
    const project = {
      name: data.name,
      description: data.description,
      createdAt: new Date().toISOString(),
      currentVersion: null,
      status: 'DISCOVERY', // DISCOVERY | SPECIFICATION | VALIDATION | ARCHITECTURE | ACTIVE
      lastSessionAt: new Date().toISOString(),
    };
    await this.fs.writeJSON(this.fs.paths.project(), project);
    return project;
  }

  // Lire project.json
  async getProject() {
    return this.fs.readJSON(this.fs.paths.project());
  }

  // Mettre à jour des champs spécifiques de project.json
  async updateProject(updates) {
    const current = await this.getProject();
    const updated = { ...current, ...updates, lastSessionAt: new Date().toISOString() };
    await this.fs.writeJSON(this.fs.paths.project(), updated);
    return updated;
  }

  // vision.md
  async getVision() {
    return this.fs.readMarkdown(this.fs.paths.vision());
  }
  async saveVision(content) {
    await this.fs.writeMarkdown(this.fs.paths.vision(), content);
  }

  // features.json
  async getFeatures() {
    return this.fs.readJSON(this.fs.paths.features());
  }
  async saveFeatures(features) {
    await this.fs.writeJSON(this.fs.paths.features(), features);
  }

  // tech-stack.json
  async getTechStack() {
    return this.fs.readJSON(this.fs.paths.techStack());
  }
  async saveTechStack(stack) {
    await this.fs.writeJSON(this.fs.paths.techStack(), stack);
  }

  // Version meta
  async getVersionMeta(version) {
    return this.fs.readJSON(this.fs.paths.versionMeta(version));
  }
  async saveVersionMeta(version, meta) {
    await this.fs.writeJSON(this.fs.paths.versionMeta(version), meta);
  }

  // Progress d'une version
  async getProgress(version) {
    const progress = await this.fs.readJSON(this.fs.paths.versionProgress(version));
    return progress ?? { done: [], pending: [], failed: [], deferred: [] };
  }
  async saveProgress(version, progress) {
    await this.fs.writeJSON(this.fs.paths.versionProgress(version), progress);
  }

  // Version active
  async getActiveVersion() {
    const project = await this.getProject();
    return project?.currentVersion ?? null;
  }

  // Timestamp de la dernière session (pour SyncChecker)
  async getLastSessionTimestamp() {
    const project = await this.getProject();
    return project?.lastSessionAt ? new Date(project.lastSessionAt) : null;
  }

  // Résumé court pour ContextManager (~500 tokens max)
  async getProjectSummary() {
    const project = await this.getProject();
    const stack = await this.getTechStack();
    if (!project) return null;
    return {
      name: project.name,
      description: project.description.substring(0, 200),
      status: project.status,
      currentVersion: project.currentVersion,
      language: stack?.language ?? 'unknown',
      framework: stack?.framework ?? 'unknown',
    };
  }

  // Lister toutes les versions
  async listVersions() {
    return this.fs.listVersions();
  }

  // failure-patterns.json — lu/écrit par FailurePatterns.js, exposé ici pour accès unifié
  async getFailurePatterns() {
    return this.fs.readJSON(this.fs.paths.failurePatterns()) ?? [];
  }
  async saveFailurePatterns(patterns) {
    await this.fs.writeJSON(this.fs.paths.failurePatterns(), patterns);
  }

  // design.json — préférences visuelles collectées en DiscoveryPhase
  // { style, colorScheme, mood, references, keywords, customNotes }
  async getDesign() {
    return this.fs.readJSON(this.fs.paths.design());
  }
  async saveDesign(design) {
    await this.fs.writeJSON(this.fs.paths.design(), design);
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `initProject()` crée tous les fichiers de base + dossiers `questions/` et `briefings/` | ⬜ |
| 2 | `getProjectSummary()` retourne moins de 500 tokens (vérifier en test) | ⬜ |
| 3 | `updateProject()` met à jour `lastSessionAt` automatiquement | ⬜ |
| 4 | `getProgress()` retourne la structure vide si fichier absent | ⬜ |
| 5 | `getFailurePatterns()` retourne un tableau vide si le fichier n'existe pas encore | ⬜ |
| 6 | Tests unitaires couvrent init + lecture + écriture | ⬜ |
| 7 | `getLastSessionTimestamp()` retourne `null` si `lastSessionAt` est absent de `project.json` | ⬜ |
| 8 | `initProject()` inclut `lastSessionAt` dans le schéma initial de `project.json` | ⬜ |
| 9 | `getDesign()` retourne `null` si `design.json` n'existe pas encore | ⬜ |
| 10 | `saveDesign()` persiste correctement les préférences visuelles dans `design.json` | ⬜ |
