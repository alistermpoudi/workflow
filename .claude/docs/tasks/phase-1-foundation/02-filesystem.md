# Phase 1 — Tâche 1.2 : FileSystem.js

## Objectif

Créer le module `FileSystem.js` qui encapsule toutes les opérations sur les fichiers `.workflow/`. C'est la couche basse sur laquelle tous les autres modules s'appuient. Toutes les opérations doivent être async et gérer les erreurs proprement.

## Fichiers à Créer / Modifier

- `src/tools/FileSystem.js` [CRÉER]
- `tests/unit/FileSystem.test.js` [CRÉER]

## Responsabilités

- Initialiser la structure `.workflow/` dans un projet cible
- Lire/écrire des fichiers JSON (project.json, progress.json, etc.)
- Lire/écrire des fichiers Markdown (vision.md, TASK-XXX.md, decisions.log)
- Vérifier l'existence de fichiers et dossiers
- Lister les fichiers de tâches d'une version
- Opérations atomiques (écrire dans un fichier temporaire puis renommer)

## Implémentation

```javascript
// src/tools/FileSystem.js
import { readFile, writeFile, mkdir, access, readdir, rename } from 'fs/promises';
import { join, dirname } from 'path';
import { constants } from 'fs';

export class FileSystem {
  constructor(projectRoot) {
    this.projectRoot = projectRoot;
    this.workflowDir = join(projectRoot, '.workflow');
  }

  // Chemins standards
  paths = {
    project: () => join(this.workflowDir, 'project.json'),
    vision: () => join(this.workflowDir, 'vision.md'),
    features: () => join(this.workflowDir, 'features.json'),
    techStack: () => join(this.workflowDir, 'tech-stack.json'),
    codeIndex: () => join(this.workflowDir, 'code-index.json'),
    decisionsLog: () => join(this.workflowDir, 'decisions.log'),
    versionDir: (v) => join(this.workflowDir, 'versions', v),
    versionMeta: (v) => join(this.workflowDir, 'versions', v, 'meta.json'),
    versionProgress: (v) => join(this.workflowDir, 'versions', v, 'progress.json'),
    taskFile: (v, taskId) => join(this.workflowDir, 'versions', v, 'tasks', `${taskId}.md`),
    tasksDir: (v) => join(this.workflowDir, 'versions', v, 'tasks'),
  };

  // Initialiser la structure .workflow/ complète
  async init() {
    await mkdir(this.workflowDir, { recursive: true });
    await mkdir(join(this.workflowDir, 'versions'), { recursive: true });
  }

  // Vérifier si .workflow/ existe
  async isInitialized() {
    try {
      await access(this.workflowDir, constants.F_OK);
      return true;
    } catch {
      return false;
    }
  }

  // Lire un fichier JSON — retourne null si absent
  async readJSON(filePath) {
    try {
      const content = await readFile(filePath, 'utf-8');
      return JSON.parse(content);
    } catch (err) {
      if (err.code === 'ENOENT') return null;
      throw err;
    }
  }

  // Écrire un fichier JSON (atomique : tmp → rename)
  async writeJSON(filePath, data) {
    const dir = dirname(filePath);
    await mkdir(dir, { recursive: true });
    const tmp = `${filePath}.tmp`;
    await writeFile(tmp, JSON.stringify(data, null, 2), 'utf-8');
    await rename(tmp, filePath);
  }

  // Lire un fichier Markdown — retourne null si absent
  async readMarkdown(filePath) {
    try {
      return await readFile(filePath, 'utf-8');
    } catch (err) {
      if (err.code === 'ENOENT') return null;
      throw err;
    }
  }

  // Écrire un fichier Markdown (atomique)
  async writeMarkdown(filePath, content) {
    const dir = dirname(filePath);
    await mkdir(dir, { recursive: true });
    const tmp = `${filePath}.tmp`;
    await writeFile(tmp, content, 'utf-8');
    await rename(tmp, filePath);
  }

  // Lire sélectivement un ensemble de fichiers source (pour ContextManager)
  async readSelective(filePaths) {
    const results = {};
    await Promise.all(
      filePaths.map(async (fp) => {
        try {
          results[fp] = await readFile(join(this.projectRoot, fp), 'utf-8');
        } catch {
          results[fp] = null; // Fichier non encore créé
        }
      })
    );
    return results;
  }

  // Lister les tâches d'une version (retourne les IDs triés : TASK-001, TASK-002...)
  async listTaskIds(version) {
    const dir = this.paths.tasksDir(version);
    try {
      const files = await readdir(dir);
      return files
        .filter(f => f.endsWith('.md'))
        .map(f => f.replace('.md', ''))
        .sort();
    } catch {
      return [];
    }
  }

  // Lister les versions disponibles
  async listVersions() {
    const dir = join(this.workflowDir, 'versions');
    try {
      const entries = await readdir(dir, { withFileTypes: true });
      return entries
        .filter(e => e.isDirectory())
        .map(e => e.name)
        .sort();
    } catch {
      return [];
    }
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `init()` crée la structure `.workflow/versions/` | ⬜ |
| 2 | `readJSON` retourne `null` pour un fichier absent (pas d'exception) | ⬜ |
| 3 | `writeJSON` est atomique (passe par un fichier `.tmp`) | ⬜ |
| 4 | `readSelective` lit plusieurs fichiers en parallèle | ⬜ |
| 5 | `listTaskIds` retourne les IDs triés | ⬜ |
| 6 | Tests unitaires couvrent les cas normaux et les fichiers absents | ⬜ |
