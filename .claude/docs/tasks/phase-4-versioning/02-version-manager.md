# Phase 4 — Tâche 4.2 : VersionManager.js + GitManager.js (complet)

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

// Créer et switcher vers une branche
async createBranch(branch) {
  await this.run(`git checkout -b ${branch}`);
}

// Switcher vers une branche existante
async checkout(branch) {
  await this.run(`git checkout ${branch}`);
}

// Merger une branche dans la courante
async merge(branch) {
  await this.run(`git merge ${branch} --no-ff -m "Merge ${branch}"`);
}

// Commiter tous les changements
async commit(message) {
  await this.run('git add -A');
  await this.run(`git commit -m "${message.replace(/"/g, '\\"')}"`);
}

// Tag d'une version
async tag(version) {
  await this.run(`git tag -a ${version} -m "Version ${version}"`);
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
  async create(name, description) {
    const versions = await this.memory.listVersions();
    if (versions.includes(name)) {
      throw new Error(`La version "${name}" existe déjà.`);
    }

    // Créer la branche Git
    const branch = `workflow/${name}`;
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

  // Switcher vers une version (bloque si repo non propre)
  async switch(version) {
    const check = await this.sync.checkBeforeSwitch();
    if (!check.canSwitch) {
      this.io.warn(check.message);
      this.io.display('Commite tes changements puis relance `workflow version switch ' + version + '`');
      return { switched: false };
    }

    const meta = await this.memory.getVersionMeta(version);
    if (!meta) throw new Error(`Version "${version}" introuvable.`);

    await this.git.checkout(meta.branch);
    await this.memory.updateProject({ currentVersion: version });

    this.io.success(`Switché vers ${version} (${meta.branch}).`);
    return { switched: true, version, branch: meta.branch };
  }

  // Marquer la version active comme complétée
  async complete() {
    const activeVersion = await this.memory.getActiveVersion();
    if (!activeVersion) throw new Error('Aucune version active.');

    const progress = await this.memory.getProgress(activeVersion);
    if (progress.pending.length > 0) {
      const confirm = await this.io.confirm(
        `⚠️ ${progress.pending.length} tâche(s) en attente. Compléter quand même ?`
      );
      if (!confirm) return;
    }

    const meta = await this.memory.getVersionMeta(activeVersion);

    // Merger dans main
    await this.git.checkout('main');
    await this.git.merge(meta.branch);
    await this.git.tag(activeVersion);

    // Mettre à jour le statut
    await this.memory.saveVersionMeta(activeVersion, { ...meta, status: STATUS.COMPLETED });
    await this.memory.updateProject({ currentVersion: null });

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
| 6 | `hotfix` crée la branche depuis la branche parente (pas main) | ⬜ |
| 7 | Une seule version peut être ACTIVE à la fois | ⬜ |
