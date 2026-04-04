# Phase 4 — Tâche 4.2 : VersionManager.js + GitManager.js (complet)

> **Note** : Ce dossier s'appelle `phase-4-versioning` pour des raisons historiques.
> Il correspond à la **Phase 4 — MCP Server (Workflow Core)** dans CLAUDE.md.

## Objectif

Implémenter `VersionManager.js` — le gestionnaire complet du cycle de vie des versions. Il couple les versions Workflow aux branches Git et applique les règles strictes (no stash, une seule version ACTIVE, etc.). Cette tâche complète aussi `GitManager.js` avec les commandes d'écriture (checkout, merge, branch).

## Dépendances

- Tâche 1.6 ✅ (GitManager read-only)
- Phase 3 ✅

## Fichiers à Créer / Modifier

- `src/core/VersionManager.js` [CRÉER]
- `src/tools/GitManager.js` [MODIFIER — ajouter les commandes d'écriture]

## Compléter `GitManager.js` — Commandes d'Écriture

```javascript
// À ajouter dans src/tools/GitManager.js

// Créer et switcher vers une branche (sûr — nom de branche passé via execFile)
async createBranch(branch) {
  return this.runSafe('git', ['checkout', '-b', branch]);
}

// Switcher vers une branche existante (sûr)
async checkout(branch) {
  return this.runSafe('git', ['checkout', branch]);
}

// Merger une branche dans la courante (sûr)
async merge(branch) {
  return this.runSafe('git', ['merge', '--no-ff', branch]);
}

// Commiter tous les changements (sûr — message passé via execFile, sans interprétation shell)
async commit(message) {
  await this.runSafe('git', ['add', '-A']);
  return this.runSafe('git', ['commit', '-m', message]);
}

// Tag d'une version (sûr)
async tag(version) {
  return this.runSafe('git', ['tag', '-a', version, '-m', `Version ${version}`]);
}

// Détecter la branche par défaut (main ou master ou autre)
async getDefaultBranch() {
  try {
    // Essayer d'abord via remote tracking
    const result = await this.run('git symbolic-ref refs/remotes/origin/HEAD');
    return result.replace('refs/remotes/origin/', '');
  } catch {
    // Fallback : chercher main ou master localement
    try {
      await this.run('git rev-parse --verify main');
      return 'main';
    } catch {
      return 'master';
    }
  }
}
```

## `VersionManager.js`

```javascript
// src/core/VersionManager.js
import { ProjectMemory } from './ProjectMemory.js';
import { GitManager } from '../tools/GitManager.js';
import { SyncChecker } from './SyncChecker.js';
import { TaskManager } from '../tools/TaskManager.js';

const STATUS = {
  DRAFT: 'DRAFT',
  ACTIVE: 'ACTIVE',
  COMPLETED: 'COMPLETED',
  ARCHIVED: 'ARCHIVED',
};

export class VersionManager {
  constructor(projectRoot, io) {
    this.memory = new ProjectMemory(projectRoot);
    this.git = new GitManager(projectRoot);
    this.sync = new SyncChecker(projectRoot);
    this.tasks = new TaskManager(projectRoot);
    this.io = io;
  }

  // Lister les versions avec leur statut
  async list() {
    const versions = await this.memory.listVersions();
    const result = [];
    for (const v of versions) {
      const meta = await this.memory.getVersionMeta(v);
      const progress = await this.memory.getProgress(v);
      result.push({
        version: v,
        status: meta?.status ?? 'UNKNOWN',
        title: meta?.title ?? '',
        tasks: { done: progress.done.length, pending: progress.pending.length },
      });
    }

    this.io.header('Versions');
    result.forEach(v => {
      const icon = { DRAFT: '📝', ACTIVE: '🟢', COMPLETED: '✅', ARCHIVED: '📦' }[v.status] ?? '?';
      this.io.display(`  ${icon} ${v.version} [${v.status}] — ${v.tasks.done}/${v.tasks.done + v.tasks.pending} tâches`);
    });

    return result;
  }

  // Créer une nouvelle version
  // Cas nominal : version absente → crée meta.json + branche Git
  // Cas post-ValidationPhase : version déjà dans .workflow/ mais branche Git absente → crée seulement la branche
  async create(name, description) {
    const versions = await this.memory.listVersions();
    const branch = `workflow/${name}`;

    if (versions.includes(name)) {
      // La version existe dans .workflow/ — vérifier si la branche Git est aussi là
      const branchExists = await this.git.branchExists(branch);
      if (branchExists) {
        throw new Error(
          `La version "${name}" existe déjà (meta.json + branche Git).\n` +
          `Utilise \`workflow version switch ${name}\` pour l'activer.`
        );
      }
      // Branche absente — créer uniquement la branche (ValidationPhase a déjà créé le meta.json)
      await this.git.createBranch(branch);
      this.io.display(`→ git checkout -b ${branch}`);
      this.io.success(`Branche "${branch}" créée pour la version "${name}" existante.`);
      return { version: name, branch };
    }

    // Créer la branche Git
    const branchExists = await this.git.branchExists(branch);
    if (!branchExists) {
      await this.git.createBranch(branch);
      this.io.display(`→ git checkout -b ${branch}`);
    }

    // Créer meta.json
    await this.memory.saveVersionMeta(name, {
      title: name,
      description: description ?? '',
      status: STATUS.DRAFT,
      branch,
      createdAt: new Date().toISOString(),
      type: 'RELEASE',
    });

    this.io.success(`Version "${name}" créée (branche: ${branch}).`);
    return { version: name, branch };
  }

  // Switcher vers une version (bloque si repo non propre ou si version active non complétée)
  async switch(version) {
    // 1. Vérifier que le repo est propre
    const check = await this.sync.checkBeforeSwitch();
    if (!check.canSwitch) {
      this.io.warn(check.message);
      this.io.display('Commite tes changements puis relance `workflow version switch ' + version + '`');
      return { success: false, message: check.message };
    }

    // 2. Vérifier que la version cible existe
    const meta = await this.memory.getVersionMeta(version);
    if (!meta) return { success: false, message: `Version "${version}" introuvable.` };

    // 3. Vérifier la règle "1 version ACTIVE à la fois"
    const currentVersion = await this.memory.getActiveVersion();
    if (currentVersion && currentVersion !== version) {
      const currentMeta = await this.memory.getVersionMeta(currentVersion);
      if (currentMeta?.status === 'ACTIVE') {
        // Autoriser seulement si la version cible est un hotfix
        const isHotfix = meta.type === 'HOTFIX';
        if (!isHotfix) {
          const message =
            `Impossible de switcher vers ${version} : la version ${currentVersion} est toujours ACTIVE.\n` +
            `Complète ${currentVersion} avec "workflow version complete" avant de changer de version.\n` +
            `Exception : les hotfixes peuvent interrompre une version ACTIVE.`;
          this.io.warn(message);
          return { success: false, message };
        }
      }
    }

    // 4. Effectuer le switch + passer la version à ACTIVE
    await this.git.checkout(meta.branch);
    await this.memory.saveVersionMeta(version, { ...meta, status: STATUS.ACTIVE });
    await this.memory.updateProject({ currentVersion: version });

    this.io.success(`Switché vers ${version} (${meta.branch}).`);
    return { success: true, version, branch: meta.branch };
  }

  // Marquer la version active comme complétée
  // options.skipConfirmation = true pour bypasser la confirmation (mode MCP avec force: true)
  async complete(options = {}) {
    const activeVersion = await this.memory.getActiveVersion();
    if (!activeVersion) throw new Error('Aucune version active.');

    const progress = await this.memory.getProgress(activeVersion);
    if (progress.pending.length > 0 && options.skipConfirmation !== true) {
      const confirm = await this.io.confirm(
        `⚠️ ${progress.pending.length} tâche(s) en attente. Compléter quand même ?`
      );
      if (!confirm) return;
    }

    const meta = await this.memory.getVersionMeta(activeVersion);

    // Merger dans la branche par défaut (main ou master)
    const defaultBranch = await this.git.getDefaultBranch();
    await this.git.checkout(defaultBranch);
    await this.git.merge(meta.branch);
    await this.git.tag(activeVersion);

    // Mettre à jour le statut
    await this.memory.saveVersionMeta(activeVersion, { ...meta, status: STATUS.COMPLETED });
    await this.memory.updateProject({ currentVersion: null });

    // Fusionner les failure-patterns du projet vers la bibliothèque globale (Phase 6)
    // await WorkflowLibrary.mergeProjectFailures(this.projectRoot);

    // Générer le CHANGELOG automatiquement (Phase 6)
    // await DocGenerator.generateChangelog(this.projectRoot, activeVersion);

    // Afficher le bilan
    await this.displayCompletionSummary(activeVersion, progress);

    return { completed: true, version: activeVersion };
  }

  // Créer un hotfix sur une version précédente
  async hotfix(name, reason) {
    const check = await this.sync.checkBeforeSwitch();
    if (!check.canSwitch) {
      this.io.warn(check.message);
      this.io.display('Commite tes changements avant de créer un hotfix.');
      return;
    }

    const [, parentVersion] = name.match(/^(v\d+\.\d+)/) ?? [];
    if (!parentVersion) throw new Error(`Nom de hotfix invalide. Exemple: v1.0.1`);

    const parentMeta = await this.memory.getVersionMeta(parentVersion);
    if (!parentMeta) throw new Error(`Version parente "${parentVersion}" introuvable.`);

    // Créer la branche hotfix depuis la branche parente
    const hotfixBranch = `workflow/hotfix/${name}`;
    await this.git.checkout(parentMeta.branch);
    await this.git.createBranch(hotfixBranch);

    await this.memory.saveVersionMeta(name, {
      title: name,
      description: reason,
      status: STATUS.ACTIVE,
      branch: hotfixBranch,
      createdAt: new Date().toISOString(),
      type: 'HOTFIX',
      parent: parentVersion,
    });

    await this.memory.updateProject({ currentVersion: name });

    this.io.success(`Hotfix ${name} créé depuis ${parentVersion}. Branche: ${hotfixBranch}`);
    this.io.display('Génère la tâche de correction, corrige, puis `workflow version complete`.');

    const backport = await this.io.confirm(`Veux-tu aussi intégrer ce fix dans la version active ?`);
    return { hotfix: name, branch: hotfixBranch, suggestBackport: backport };
  }

  // Retourner l'état complet d'une version (ou de la version active si non spécifiée)
  async status(version = null) {
    const targetVersion = version ?? await this.memory.getActiveVersion();
    if (!targetVersion) return { error: 'Aucune version active' };

    const meta = await this.memory.getVersionMeta(targetVersion);
    const progress = await this.memory.getProgress(targetVersion);

    return {
      version: targetVersion,
      status: meta?.status ?? 'UNKNOWN',
      branch: meta?.branch ?? null,
      done: progress.done.length,
      pending: progress.pending.length,
      deferred: progress.deferred?.length ?? 0,
      tasks: {
        done: progress.done,
        pending: progress.pending,
      },
    };
  }

  async displayCompletionSummary(version, progress) {
    this.io.header(`Bilan ${version}`);
    this.io.display(`✅ ${progress.done.length} tâches complétées`);
    if (progress.deferred?.length > 0) {
      this.io.display(`⚠️  ${progress.deferred.length} tâches reportées :`);
      progress.deferred.forEach(d =>
        this.io.display(`   → ${d.id} vers ${d.to} — ${d.reason}`)
      );
    }
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `create` crée la branche Git et le `meta.json` | ⬜ |
| 2 | `switch` bloque avec message explicite si repo non propre | ⬜ |
| 3 | `switch` fait le `git checkout` ET met à jour `currentVersion` dans `project.json` | ⬜ |
| 4 | `complete` merge dans main et tag la version | ⬜ |
| 5 | `complete` affiche le bilan avec tâches reportées | ⬜ |
| 6 | `complete` contient les appels commentés `WorkflowLibrary.mergeProjectFailures()` et `DocGenerator.generateChangelog()` (Phase 6) | ⬜ |
| 7 | `hotfix` crée la branche depuis la branche parente (pas main) | ⬜ |
| 8 | Une seule version peut être ACTIVE à la fois | ⬜ |
| 9 | `status()` sans argument retourne l'état de la version active | ⬜ |
| 10 | `status('v1.0')` retourne l'état de v1.0 même si une autre version est active | ⬜ |
| 11 | `switch('v1.5')` est refusé si v1.0 est encore ACTIVE (message explicite) | ⬜ |
| 12 | `switch('hotfix/v1.0.1')` est autorisé même si v1.5 est ACTIVE (type HOTFIX) | ⬜ |
| 13 | `switch('v1.5')` est autorisé si v1.0 est COMPLETED | ⬜ |
| 14 | `complete()` fonctionne sur un repo avec branche par défaut `master` | ⬜ |
| 15 | `complete()` fonctionne sur un repo avec branche par défaut `main` | ⬜ |
| 16 | `complete({ skipConfirmation: true })` ne demande pas de confirmation | ⬜ |
| 17 | `complete()` sans `skipConfirmation` demande bien la confirmation avant de procéder | ⬜ |
| 18 | `switch(version)` met à jour le statut de la version à `ACTIVE` dans `meta.json` | ⬜ |
| 19 | `switch('v1.5')` est refusé si une autre version existe avec `status: ACTIVE` (règle appliquée réellement) | ⬜ |
| 20 | `create('v1.0')` après `ValidationPhase` ne throw pas si meta.json existe mais branche absente | ⬜ |
| 21 | `create('v1.0')` throw si meta.json ET branche Git existent déjà (message explicite avec `workflow version switch`) | ⬜ |
