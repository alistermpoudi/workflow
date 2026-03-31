# Phase 1 — Tâche 1.5 : DecisionsLog.js

## Objectif

Créer le module `DecisionsLog.js` — le journal actif des décisions techniques. C'est l'un des composants les plus importants de Workflow : il permet de ne jamais reproposer une solution déjà écartée et de maintenir la cohérence des choix techniques à travers les sessions.

## Dépendances

- Tâche 1.2 ✅ (`FileSystem.js`)

## Fichiers à Créer / Modifier

- `src/core/DecisionsLog.js` [CRÉER]
- `tests/unit/DecisionsLog.test.js` [CRÉER]

## Format du `decisions.log`

```
[2025-03-12] [TASK-004] ORM : Prisma choisi plutôt que TypeORM
  Décision : utiliser Prisma
  Raison : meilleure DX, migrations plus fiables sur PostgreSQL

[2025-03-14] [TASK-006] Base de données : pas d'Entity Framework
  Décision : utiliser des fonctions stockées PostgreSQL
  Raison : performance critique sur les requêtes de facturation

[2025-03-18] [TASK-008] Auth : JWT uniquement, pas de sessions serveur
  Décision : JWT stateless
  Raison : architecture stateless requise pour déploiement serverless
```

Le format est du **texte brut** — lisible par un humain sans outil spécial. Chaque entrée est séparée par une ligne vide.

## Implémentation

```javascript
// src/core/DecisionsLog.js
import { readFile, appendFile, mkdir } from 'fs/promises';
import { dirname } from 'path';
import { FileSystem } from '../tools/FileSystem.js';

export class DecisionsLog {
  constructor(projectRoot) {
    this.fs = new FileSystem(projectRoot);
    this.logPath = this.fs.paths.decisionsLog();
  }

  // Enregistrer une nouvelle décision
  async log(taskId, decision, reason) {
    const date = new Date().toISOString().split('T')[0];
    const entry = `[${date}] [${taskId}] ${decision}\n  Décision : ${decision}\n  Raison : ${reason}\n\n`;

    await mkdir(dirname(this.logPath), { recursive: true });
    await appendFile(this.logPath, entry, 'utf-8');
  }

  // Enregistrer plusieurs décisions d'une traite (après une tâche)
  async logMany(taskId, decisions) {
    for (const d of decisions) {
      await this.log(taskId, d.decision, d.reason);
    }
  }

  // Lire les N dernières entrées (pour ContextManager)
  async getRecent(n = 5) {
    const content = await this.readRaw();
    if (!content) return [];
    return this.parseEntries(content).slice(-n);
  }

  // Obtenir les entrées pertinentes pour une tâche (recherche par mots-clés)
  // Utilisé par ExecutionLoop avant de coder une tâche
  async getRelevant(task) {
    const content = await this.readRaw();
    if (!content) return [];

    const entries = this.parseEntries(content);

    // Recherche simple par mots-clés dans le titre de la tâche et son contexte
    const keywords = this.extractKeywords(task);
    return entries.filter(entry =>
      keywords.some(kw =>
        entry.raw.toLowerCase().includes(kw.toLowerCase())
      )
    );
  }

  // Lire le contenu brut
  async readRaw() {
    try {
      return await readFile(this.logPath, 'utf-8');
    } catch (err) {
      if (err.code === 'ENOENT') return null;
      throw err;
    }
  }

  // Parser les entrées du log
  parseEntries(content) {
    const blocks = content.split('\n\n').filter(b => b.trim());
    return blocks.map(block => {
      const lines = block.split('\n');
      const header = lines[0];
      const dateMatch = header.match(/\[(\d{4}-\d{2}-\d{2})\]/);
      const taskMatch = header.match(/\[(\w+-\d+)\]/);
      return {
        date: dateMatch?.[1] ?? '',
        taskId: taskMatch?.[1] ?? '',
        raw: block,
        summary: header,
      };
    });
  }

  // Extraire des mots-clés pertinents d'une tâche
  extractKeywords(task) {
    const technicalTerms = [
      'ORM', 'auth', 'JWT', 'session', 'database', 'DB', 'SQL',
      'API', 'REST', 'GraphQL', 'cache', 'Redis', 'migration',
      'test', 'lint', 'build', 'deploy', 'CI', 'Docker',
    ];

    const keywords = [];
    const titleWords = task.title?.split(/\s+/) ?? [];
    keywords.push(...titleWords.filter(w => w.length > 3));
    keywords.push(...technicalTerms.filter(t =>
      task.context?.includes(t) || task.title?.includes(t)
    ));

    return [...new Set(keywords)];
  }
}
```

## Utilisation par `ExecutionLoop` (preview Phase 3)

```javascript
// Dans ExecutionPhase.js — avant de coder une tâche
const relevantDecisions = await decisionsLog.getRelevant(task);
if (relevantDecisions.length > 0) {
  console.log('Décisions antérieures pertinentes :');
  relevantDecisions.forEach(d => console.log(' -', d.summary));
}
// → Ces décisions sont injectées dans le prompt LLM
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `log()` appende une entrée formatée correctement | ⬜ |
| 2 | `getRecent(5)` retourne les 5 dernières entrées | ⬜ |
| 3 | `getRelevant(task)` trouve les entrées liées par mots-clés | ⬜ |
| 4 | `parseEntries` parse correctement le format texte | ⬜ |
| 5 | Retourne `[]` (pas une exception) si `decisions.log` n'existe pas encore | ⬜ |
| 6 | Tests unitaires couvrent log + lecture + pertinence | ⬜ |
