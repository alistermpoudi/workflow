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
import { exec, execFile as execFileCb } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);
const execFileAsync = promisify(execFileCb);

export class GitManager {
  constructor(projectRoot) {
    this.projectRoot = projectRoot;
  }

  // run() reste pour les commandes read-only sans données utilisateur
  async run(cmd) {
    const { stdout } = await execAsync(cmd, { cwd: this.projectRoot });
    return stdout.trim();
  }

  // runSafe() pour les commandes avec données utilisateur — utilise execFile
  async runSafe(bin, args) {
    const { stdout } = await execFileAsync(bin, args, { cwd: this.projectRoot });
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

  // Vérifier si une branche existe (sûr — branch passé via execFile, pas d'injection shell)
  async branchExists(branch) {
    try {
      await this.runSafe('git', ['rev-parse', '--verify', branch]);
      return true;
    } catch {
      return false;
    }
  }

  // Lister les fichiers modifiés non-commités (staged + unstaged, hors .workflow/)
  async getUncommittedFiles() {
    const result = await this.run('git status --porcelain');
    if (!result) return [];
    return result
      .split('\n')
      .filter(Boolean)
      .map(line => line.slice(3).trim()) // Retirer le code de statut (ex: " M ", "?? ")
      .filter(f => !f.startsWith('.workflow/')); // Ignorer les fichiers .workflow/
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
import { join } from 'path';
import { GitManager } from '../tools/GitManager.js';
import { ProjectMemory } from './ProjectMemory.js';
import { FileSystem } from '../tools/FileSystem.js';

export class SyncChecker {
  constructor(projectRoot) {
    this.projectRoot = projectRoot;
    this.git = new GitManager(projectRoot);
    this.memory = new ProjectMemory(projectRoot);
    this.fs = new FileSystem(projectRoot); // requis par checkPreconditions
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
    // Lire meta.branch depuis .workflow/ — les hotfixes ont 'workflow/hotfix/vX.Y.Z', pas 'workflow/vX.Y.Z'
    const activeVersion = project.currentVersion;
    if (activeVersion) {
      const currentBranch = await this.git.currentBranch();
      const versionMeta = await this.memory.getVersionMeta(activeVersion);
      const expectedBranch = versionMeta?.branch ?? `workflow/${activeVersion}`;

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

    // 2. Détecter modifications : commits récents ET fichiers non-commités
    // getModifiedSince() couvre les commits ; getUncommittedFiles() couvre les modifs non-commitées
    const lastSession = await this.memory.getLastSessionTimestamp();
    const committedChanges = lastSession
      ? await this.git.getModifiedSince(lastSession)
      : [];
    const uncommittedChanges = await this.git.getUncommittedFiles();
    const modifiedFiles = [...new Set([...committedChanges, ...uncommittedChanges])];

    if (modifiedFiles.length > 0) {
      const diff = await this.git.getDiff(modifiedFiles);
      return {
        type: 'MANUAL_CHANGES',
        message: `${modifiedFiles.length} fichier(s) modifié(s) depuis la dernière session.`,
        files: modifiedFiles,
        diff,
        lastSession: lastSession?.toISOString() ?? null,
        activeVersion: project.currentVersion ?? null, // Requis par WorkflowAgent._persistManualChanges()
      };
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

  // Vérifier les préconditions déclaratives d'une tâche avant de la démarrer
  // Les préconditions sont définies dans le champ "## Préconditions" de TASK-XXX.md
  async checkPreconditions(task, version) {
    const preconditions = task.preconditions;
    if (!preconditions) return { met: true };

    const failures = [];

    // Vérifier les fichiers requis
    if (preconditions.filesExist) {
      for (const fp of preconditions.filesExist) {
        const exists = await this.fs.exists(join(this.projectRoot, fp));
        if (!exists) failures.push(`Fichier requis absent : ${fp}`);
      }
    }

    // Vérifier que les tests passent
    if (preconditions.testsPass) {
      const isClean = await this.git.isClean();
      if (!isClean) failures.push('Des modifications non commitées existent — tests non fiables');
    }

    // Vérifier la branche Git
    if (preconditions.branch) {
      const currentBranch = await this.git.currentBranch();
      if (currentBranch !== preconditions.branch) {
        failures.push(`Branche requise : "${preconditions.branch}", actuelle : "${currentBranch}"`);
      }
    }

    // Vérifier les tâches dépendantes
    if (preconditions.tasksCompleted) {
      const progress = await this.memory.getProgress(version);
      for (const depId of preconditions.tasksCompleted) {
        if (!progress.done.includes(depId)) {
          failures.push(`Tâche dépendante non terminée : ${depId}`);
        }
      }
    }

    return failures.length === 0
      ? { met: true }
      : { met: false, failures };
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

## Préconditions Déclaratives dans les Tâches

Le champ `## Préconditions` est ajouté au format `TASK-XXX.md`. Il est vérifié par `SyncChecker.checkPreconditions()` avant de démarrer une tâche.

```markdown
## Préconditions
- filesExist: ["src/tools/FileSystem.js", "src/core/ProjectMemory.js"]
- tasksCompleted: ["TASK-001", "TASK-002"]
- branch: "workflow/v1.0"
- testsPass: true
```

Ce contrat est généré par `ValidationPhase` lors de la création de la tâche, à partir des dépendances déclarées. Si une précondition échoue, Workflow bloque avec un message explicite — pas d'exécution partielle silencieuse.

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `check()` retourne `NOT_INITIALIZED` si pas de `.workflow/` | ⬜ |
| 2 | `check()` retourne `BRANCH_MISMATCH` si branche incorrecte | ⬜ |
| 3 | `check()` retourne `MANUAL_CHANGES` avec la liste des fichiers | ⬜ |
| 3b | `check()` retourne `MANUAL_CHANGES` même si les fichiers sont modifiés mais non-commités | ⬜ |
| 4 | `check()` retourne `CLEAN` si tout est cohérent | ⬜ |
| 5 | `checkBeforeSwitch()` bloque si repo non propre | ⬜ |
| 6 | `GitManager.isGitRepo()` retourne false hors d'un repo | ⬜ |
| 7 | `checkPreconditions()` retourne les échecs précis pour chaque précondition | ⬜ |
| 8 | `checkPreconditions()` retourne `{ met: true }` si toutes les préconditions passent | ⬜ |
| 9 | Tests unitaires mockent les commandes git | ⬜ |
| 10 | `GitManager.commit()` utilise `execFile` (pas `exec`) — aucune injection shell possible | ⬜ |
| 11 | Un message contenant `$(rm -rf /)` est passé littéralement à git, sans interprétation shell | ⬜ |
| 12 | `GitManager.branchExists()` utilise `runSafe()` — nom de branche non interpolé dans une commande shell | ⬜ |
| 13 | `check()` utilise `meta.branch` pour comparer la branche — pas de faux MISMATCH sur les hotfixes | ⬜ |
| 14 | `check()` retourne `activeVersion` dans le rapport `MANUAL_CHANGES` | ⬜ |
