# Phase 6 — Tâche 6.4 : WorkflowLibrary.js

## Objectif

Créer la `WorkflowLibrary` — une mémoire locale **cross-projet** qui capitalise les patterns validés et les erreurs connues à travers tous les projets gérés par Workflow. Plus Workflow est utilisé, plus il devient précis sur les types de projets que l'utilisateur construit.

## Dépendances

- Phase 4 (MVP) ✅
- Tâche 6.2 (`CodeIndexer`) recommandée ✅

## Fichiers à Créer

- `src/core/WorkflowLibrary.js` [CRÉER]
- `tests/unit/WorkflowLibrary.test.js` [CRÉER]

## Structure de la Librairie

Stockée dans `~/.workflow/library/` (global, hors des projets) :

```
~/.workflow/library/
├── patterns/
│   ├── jwt-auth-node.md        # Pattern validé sur N projets
│   ├── prisma-setup.md
│   ├── vitest-esm-config.md
│   └── ...
├── failure-patterns.json       # Erreurs connues cross-projet (étend le fichier par projet)
└── index.json                  # Index des patterns avec tags et usage stats
```

## Format d'un Pattern

```markdown
---
id: jwt-auth-node
title: Authentification JWT — Node.js
tags: [auth, jwt, node, express]
usedIn: 3         # Nombre de projets où ce pattern a été appliqué
successRate: 1.0  # Proportion de fois où il a fonctionné sans modification
lastUsed: 2025-03-18
---

## Stack
Node.js + Express + jsonwebtoken + bcrypt

## Files
- src/middleware/auth.js
- src/services/jwt.service.js
- src/routes/auth.routes.js

## Critères qui déclenchent ce pattern
- Tâche contient "auth" + "JWT" dans title ou intent
- Stack = Node.js + Express

## Template de tâche pré-rempli
[contenu Markdown complet de la TASK pré-remplie]

## Notes
Fonctionne sans modification sur Express 4+. Sur Fastify, adapter
le middleware (signature différente).
```

## Implémentation

```javascript
// src/core/WorkflowLibrary.js
import { readFile, writeFile, readdir, mkdir, access } from 'fs/promises';
import { join, homedir } from 'path';

const LIBRARY_DIR = join(homedir(), '.workflow', 'library');

export class WorkflowLibrary {
  constructor() {
    this.patternsDir = join(LIBRARY_DIR, 'patterns');
    this.failurePatternsPath = join(LIBRARY_DIR, 'failure-patterns.json');
    this.indexPath = join(LIBRARY_DIR, 'index.json');
  }

  // Initialiser la librairie globale (première utilisation)
  async init() {
    await mkdir(this.patternsDir, { recursive: true });
    if (!(await this._exists(this.indexPath))) {
      await writeFile(this.indexPath, JSON.stringify([]), 'utf-8');
    }
    if (!(await this._exists(this.failurePatternsPath))) {
      await writeFile(this.failurePatternsPath, JSON.stringify([]), 'utf-8');
    }
  }

  // Chercher un pattern applicable à une tâche
  async findPattern(task) {
    const index = await this._readIndex();
    const searchText = `${task.title} ${task.intent ?? ''} ${task.context ?? ''}`.toLowerCase();

    const candidates = index
      .map(entry => ({
        ...entry,
        score: this._scorePattern(entry, searchText),
      }))
      .filter(e => e.score > 0.5)
      .sort((a, b) => b.score - a.score);

    if (candidates.length === 0) return null;

    // Charger le contenu du meilleur pattern
    const best = candidates[0];
    const content = await readFile(join(this.patternsDir, `${best.id}.md`), 'utf-8');
    return { ...best, content };
  }

  // Enregistrer un pattern depuis une tâche réussie
  async savePattern(task, projectId) {
    const index = await this._readIndex();
    const existing = index.find(e => e.id === task.patternId);

    if (existing) {
      existing.usedIn++;
      existing.lastUsed = new Date().toISOString().split('T')[0];
    } else {
      const id = this._generateId(task.title);
      index.push({
        id,
        title: task.title,
        tags: this._extractTags(task),
        usedIn: 1,
        successRate: 1.0,
        lastUsed: new Date().toISOString().split('T')[0],
      });
      // Sauvegarder le fichier pattern
      await writeFile(join(this.patternsDir, `${id}.md`), this._renderPattern(task), 'utf-8');
    }

    await writeFile(this.indexPath, JSON.stringify(index, null, 2), 'utf-8');
  }

  // Fusionner les failure-patterns d'un projet avec la librairie globale
  async mergeProjectFailures(projectFailurePatternsPath) {
    const projectPatterns = JSON.parse(await readFile(projectFailurePatternsPath, 'utf-8'));
    const globalPatterns = await this._readFailurePatterns();

    for (const pp of projectPatterns) {
      const existing = globalPatterns.find(g => g.fingerprint === pp.fingerprint);
      if (existing) {
        existing.occurrences += pp.occurrences;
        existing.lastSeen = pp.lastSeen;
      } else {
        globalPatterns.push({ ...pp, source: 'project' });
      }
    }

    await writeFile(this.failurePatternsPath, JSON.stringify(globalPatterns, null, 2), 'utf-8');
  }

  // Obtenir les failure-patterns globaux pour un nouveau projet
  async getGlobalFailurePatterns() {
    return this._readFailurePatterns();
  }

  _scorePattern(entry, searchText) {
    const tags = entry.tags.join(' ').toLowerCase();
    const title = entry.title.toLowerCase();
    const combined = `${tags} ${title}`;
    const words = searchText.split(/\W+/).filter(w => w.length > 3);
    const matches = words.filter(w => combined.includes(w)).length;
    const recencyBonus = entry.usedIn > 2 ? 0.1 : 0;
    return (matches / Math.max(words.length, 1)) + recencyBonus;
  }

  _generateId(title) {
    return title.toLowerCase().replace(/[^a-z0-9]+/g, '-').slice(0, 40);
  }

  _extractTags(task) {
    const technicalTerms = ['auth', 'jwt', 'prisma', 'postgres', 'redis', 'api', 'rest', 'graphql', 'vitest', 'eslint'];
    const text = `${task.title} ${task.context ?? ''}`.toLowerCase();
    return technicalTerms.filter(t => text.includes(t));
  }

  async _readIndex() {
    try { return JSON.parse(await readFile(this.indexPath, 'utf-8')); }
    catch { return []; }
  }

  async _readFailurePatterns() {
    try { return JSON.parse(await readFile(this.failurePatternsPath, 'utf-8')); }
    catch { return []; }
  }

  // Rendre un objet tâche en Markdown de pattern (format bibliothèque)
  _renderPattern(task) {
    const id = this._generateId(task.title);
    const tags = this._extractTags(task);
    const date = new Date().toISOString().split('T')[0];
    const files = (task.files ?? []).map(f => `- ${f.path}`).join('\n') || '(aucun)';
    const criteria = (task.criteria ?? []).map(c => `- ${c}`).join('\n') || '(aucun)';

    return `---
id: ${id}
title: ${task.title}
tags: [${tags.join(', ')}]
usedIn: 1
successRate: 1.0
lastUsed: ${date}
---

## Stack
(à compléter selon la stack du projet)

## Files
${files}

## Critères qui déclenchent ce pattern
- Tâche contient "${tags[0] ?? task.title.split(' ')[0]}" dans title ou intent

## Template de tâche pré-rempli
### Title
${task.title}

### Intent
${task.intent ?? '(à compléter)'}

### Criteria
${criteria}

## Notes
Pattern extrait automatiquement depuis une tâche réussie.
`;
  }

  // Vérifier l'existence d'un fichier sans le lire entièrement
  async _exists(path) {
    try { await access(path); return true; }
    catch { return false; }
  }
}
```

## Intégration dans le Workflow Existant

**Au démarrage d'un projet** (`workflow init`) :
```javascript
const library = new WorkflowLibrary();
await library.init();
// Pré-charger les failure-patterns globaux dans le projet
const globalFailures = await library.getGlobalFailurePatterns();
await localFailurePatterns.seed(globalFailures);
```

**Lors de la génération d'une tâche** (`ValidationPhase`) :
```javascript
const pattern = await library.findPattern(task);
if (pattern) {
  cli.display(`Pattern trouvé : "${pattern.title}" (utilisé sur ${pattern.usedIn} projets)`);
  const useIt = await cli.confirm('Pré-remplir la tâche avec ce pattern ?');
  if (useIt) task = applyPattern(task, pattern);
}
```

**Après completion d'une version** (`workflow version complete`) :
```javascript
// Sauvegarder les nouveaux patterns appris + fusionner les failures
await library.mergeProjectFailures(localFailurePatternsPath);
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `init()` crée `~/.workflow/library/` si absent | ⬜ |
| 2 | `findPattern()` retourne `null` si aucun pattern avec score > 0.5 | ⬜ |
| 3 | `findPattern()` retourne le meilleur pattern si score suffisant | ⬜ |
| 4 | `savePattern()` incrémente `usedIn` si le pattern existe déjà | ⬜ |
| 5 | `mergeProjectFailures()` agrège les occurrences sans dupliquer | ⬜ |
| 6 | `getGlobalFailurePatterns()` retourne `[]` si fichier absent (pas d'exception) | ⬜ |
| 7 | Tests unitaires utilisent un répertoire temporaire (pas `~/.workflow/library/` réel) | ⬜ |
| 8 | `savePattern()` appelle `_renderPattern(task)` qui retourne un Markdown valide | ⬜ |
| 9 | `_exists()` utilise `access()` — ne lit pas le contenu du fichier | ⬜ |
