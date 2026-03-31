# Phase 1 — Tâche 1.6 : SyncChecker.js

## Objectif

Créer le module `SyncChecker.js` — la pièce qui garantit la promesse principale de Workflow : reprendre exactement où on s'était arrêté après un context overflow ou une interruption. Il vérifie la cohérence entre l'état Git et l'état `.workflow/`, et détecte les modifications manuelles effectuées hors de Workflow.

## Dépendances

- Tâche 1.2 ✅ (`FileSystem.js`)
- Tâche 1.3 ✅ (`ProjectMemory.js`)

## Fichiers à Créer / Modifier

- `src/core/SyncChecker.js` [CRÉER]
- `src/tools/GitManager.js` [CRÉER] (uniquement les commandes read-only pour cette tâche)
- `tests/unit/SyncChecker.test.js` [CRÉER]

## Implémentation — `GitManager.js` (commandes read-only)

```javascript
// src/tools/GitManager.js
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

export class GitManager {
  constructor(projectRoot) {
    this.projectRoot = projectRoot;
  }

  async run(cmd) {
    const { stdout } = await execAsync(cmd, { cwd: this.projectRoot });
    return stdout.trim();
  }

  // Branche Git courante
  async currentBranch() {
    return this.run('git rev-parse --abbrev-ref HEAD');
  }

  // Vérifier si le répertoire de travail est propre
  async isClean() {
    const status = await this.run('git status --porcelain');
    return status === '';
  }

  // Lister les fichiers modifiés depuis un timestamp
  async getModifiedSince(since) {
    const isoDate = since instanceof Date ? since.toISOString() : since;
    const result = await this.run(
      `git log --since="${isoDate}" --name-only --format="" --diff-filter=M`
    );
    return result ? result.split('\n').filter(Boolean) : [];
  }

  // Obtenir le diff d'un fichier (pour analyse sémantique par LLM)
  async getDiff(files = []) {
    if (files.length === 0) {
      return this.run('git diff HEAD');
    }
    return this.run(`git diff HEAD -- ${files.join(' ')}`);
  }

  // Vérifier si une branche existe
  async branchExists(branch) {
    try {
      await this.run(`git rev-parse --verify ${branch}`);
      return true;
    } catch {
      return false;
    }
  }

  // Vérifier si on est dans un repo Git
  async isGitRepo() {
    try {
      await this.run('git rev-parse --git-dir');
      return true;
    } catch {
      return false;
    }
  }
}
```

## Implémentation — `SyncChecker.js`

```javascript
// src/core/SyncChecker.js
import { GitManager } from '../tools/GitManager.js';
import { ProjectMemory } from './ProjectMemory.js';

export class SyncChecker {
  constructor(projectRoot) {
    this.git = new GitManager(projectRoot);
    this.memory = new ProjectMemory(projectRoot);
  }

  // Vérification complète au démarrage d'une session
  // Retourne un rapport structuré à afficher à l'utilisateur
  async check() {
    const isRepo = await this.git.isGitRepo();
    if (!isRepo) {
      return { type: 'NO_GIT', message: 'Pas de repo Git détecté dans ce projet.' };
    }

    const project = await this.memory.getProject();
    if (!project) {
      return { type: 'NOT_INITIALIZED', message: 'Aucun projet Workflow trouvé dans .workflow/' };
    }

    // 1. Vérifier cohérence branche / version active
    const activeVersion = project.currentVersion;
    if (activeVersion) {
      const currentBranch = await this.git.currentBranch();
      const expectedBranch = `workflow/${activeVersion}`;

      if (currentBranch !== expectedBranch && currentBranch !== 'main' && currentBranch !== 'master') {
        return {
          type: 'BRANCH_MISMATCH',
          message: `Branche Git "${currentBranch}" ≠ version active "${activeVersion}" (attendu: ${expectedBranch})`,
          currentBranch,
          expectedBranch,
          activeVersion,
        };
      }
    }

    // 2. Détecter modifications manuelles depuis la dernière session
    const lastSession = await this.memory.getLastSessionTimestamp();
    if (lastSession) {
      const modifiedFiles = await this.git.getModifiedSince(lastSession);
      if (modifiedFiles.length > 0) {
        const diff = await this.git.getDiff(modifiedFiles);
        return {
          type: 'MANUAL_CHANGES',
          message: `${modifiedFiles.length} fichier(s) modifié(s) depuis la dernière session.`,
          files: modifiedFiles,
          diff,
          lastSession: lastSession.toISOString(),
        };
      }
    }

    // 3. Vérifier que le répertoire est propre (uncommitted changes)
    const isClean = await this.git.isClean();
    if (!isClean) {
      return {
        type: 'DIRTY_REPO',
        message: 'Des modifications non commitées existent dans le répertoire.',
        diff: await this.git.getDiff(),
      };
    }

    return {
      type: 'CLEAN',
      message: `Projet "${project.name}" — ${activeVersion ? `v${activeVersion} ACTIVE` : 'pas de version active'}.`,
      project,
    };
  }

  // Vérifier avant un switch de version (bloquer si repo non propre)
  async checkBeforeSwitch() {
    const isClean = await this.git.isClean();
    if (!isClean) {
      const diff = await this.git.getDiff();
      return {
        canSwitch: false,
        message: 'Répertoire non propre. Commite tes changements avant de changer de version.',
        diff,
      };
    }
    return { canSwitch: true };
  }
}
```

## Message de Reprise — Exemple Complet

Quand `SyncChecker` détecte des modifications manuelles, voici le message que Workflow doit afficher (généré par LLM après analyse du diff) :

```
Workflow : "Je reprends FreelanceApp — v1.5 ACTIVE (branche workflow/v1.5).

           Modifications détectées depuis la dernière session (2025-03-20) :
           • src/controllers/auth.js — modifié
           • src/routes/auth.routes.js — modifié

           Il semblerait que tu as codé la route POST /register manuellement.
           Je marque le critère 1 de TASK-003 comme validé.
           Je reprends à partir du critère 2 (POST /login). C'est bien ça ?"
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `check()` retourne `NOT_INITIALIZED` si pas de `.workflow/` | ⬜ |
| 2 | `check()` retourne `BRANCH_MISMATCH` si branche incorrecte | ⬜ |
| 3 | `check()` retourne `MANUAL_CHANGES` avec la liste des fichiers | ⬜ |
| 4 | `check()` retourne `CLEAN` si tout est cohérent | ⬜ |
| 5 | `checkBeforeSwitch()` bloque si repo non propre | ⬜ |
| 6 | `GitManager.isGitRepo()` retourne false hors d'un repo | ⬜ |
| 7 | Tests unitaires mockent les commandes git | ⬜ |
