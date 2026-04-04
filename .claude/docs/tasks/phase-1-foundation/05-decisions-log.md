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

## Graphe de Décisions — `decisions-graph.json`

Le `decisions.log` reste lisible par l'humain. En parallèle, `decisions-graph.json` stocke les **relations entre décisions** pour permettre la détection de contradictions.

```json
{
  "DECISION-001": {
    "taskId": "TASK-004",
    "summary": "Prisma choisi comme ORM",
    "date": "2025-03-12",
    "confidence": "HIGH",
    "scope": "global",
    "revisable": false,
    "relations": [
      { "type": "DEPENDS_ON", "target": "DECISION-003", "reason": "PostgreSQL requis" },
      { "type": "CONTRADICTS", "target": "DECISION-008", "reason": "DECISION-008 interdisait tout ORM" }
    ]
  }
}
```

**Types de relations** :
- `DEPENDS_ON` — cette décision présuppose une autre
- `CONTRADICTS` — conflit direct avec une autre décision
- `SUPERSEDES` — remplace/annule une décision plus ancienne
- `REFINES` — affine une décision sans la contredire

**Niveaux de confiance** : `HIGH` (décision actée) | `MEDIUM` (à revalider) | `LOW` (expérimentale)

**`revisable: false`** — signale qu'une décision ne doit jamais être silencieusement contournée (ex: choix ORM, auth, DB principale).

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

  // Enregistrer une nouvelle décision (log texte + graphe JSON)
  async log(taskId, decision, reason, options = {}) {
    const date = new Date().toISOString().split('T')[0];
    const entry = `[${date}] [${taskId}] ${decision}\n  Décision : ${decision}\n  Raison : ${reason}\n\n`;

    await mkdir(dirname(this.logPath), { recursive: true });
    await appendFile(this.logPath, entry, 'utf-8');

    // Mettre à jour le graphe
    const graphResult = await this._addToGraph(taskId, decision, date, options);
    return graphResult; // { id, contradictions? }
  }

  // Ajouter une entrée au graphe de décisions
  async _addToGraph(taskId, summary, date, options = {}) {
    const graphPath = this.fs.paths.decisionsGraph();
    const graph = (await this.fs.readJSON(graphPath)) ?? {};

    const id = `DECISION-${String(Object.keys(graph).length + 1).padStart(3, '0')}`;
    graph[id] = {
      taskId,
      summary,
      date,
      confidence: options.confidence ?? 'HIGH',
      scope: options.scope ?? 'global',
      revisable: options.revisable ?? true,
      relations: options.relations ?? [],
    };

    await this.fs.writeJSON(graphPath, graph);

    // Détecter les contradictions immédiatement après ajout
    const contradictions = await this.checkContradictions(id, graph);
    if (contradictions.length > 0) {
      return { id, contradictions }; // Remonter pour alerte à l'utilisateur
    }
    return { id };
  }

  // Détecter les contradictions dans le graphe
  async checkContradictions(newId = null, graph = null) {
    const g = graph ?? (await this.fs.readJSON(this.fs.paths.decisionsGraph())) ?? {};
    const contradictions = [];

    const targets = newId ? [newId] : Object.keys(g);
    for (const id of targets) {
      const node = g[id];
      for (const rel of (node.relations ?? [])) {
        if (rel.type === 'CONTRADICTS' && g[rel.target]) {
          const target = g[rel.target];
          // Contradiction active seulement si les deux sont non-superseded
          const isSuperseded = Object.values(g).some(n =>
            n.relations?.some(r => r.type === 'SUPERSEDES' && (r.target === id || r.target === rel.target))
          );
          if (!isSuperseded) {
            contradictions.push({ source: id, target: rel.target, reason: rel.reason });
          }
        }
      }
    }
    return contradictions;
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

  // Obtenir TOUTES les entrées du log (utilisé par ContextManager pour le scoring)
  // Retourne un tableau d'objets conformes au schéma suivant :
  // {
  //   date: string,        // "2025-03-12"
  //   taskId: string,      // "TASK-004"
  //   summary: string,     // Première ligne du bloc (ex: "ORM : Prisma choisi")
  //   confidence: string,  // "HIGH" | "MEDIUM" | "LOW" — depuis decisions-graph.json si disponible, défaut "HIGH"
  //   scope: string,       // "global" | nom de module — depuis decisions-graph.json si disponible, défaut "global"
  //   raw: string,         // Texte brut complet du bloc
  // }
  async getAll() {
    const content = await this.readRaw();
    if (!content) return [];
    const entries = this.parseEntries(content);

    // Enrichir avec les données du graphe (confidence, scope) si disponible
    const graph = (await this.fs.readJSON(this.fs.paths.decisionsGraph())) ?? {};
    const graphByTaskId = {};
    Object.values(graph).forEach(node => {
      graphByTaskId[node.taskId] = node;
    });

    return entries.map(entry => {
      const graphNode = graphByTaskId[entry.taskId];
      return graphNode
        ? { ...entry, confidence: graphNode.confidence ?? 'HIGH', scope: graphNode.scope ?? 'global' }
        : entry;
    });
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
  // Retourne des objets avec les champs : date, taskId, summary, confidence, scope, raw
  // confidence et scope sont des valeurs par défaut — enrichis par getAll() depuis decisions-graph.json
  parseEntries(content) {
    const blocks = content.split('\n\n').filter(b => b.trim());
    return blocks.map(block => {
      const lines = block.split('\n');
      const header = lines[0];
      const dateMatch = header.match(/\[(\d{4}-\d{2}-\d{2})\]/);
      const taskMatch = header.match(/\[(\w+-\d+)\]/);
      // Extraire le résumé (tout après [TASK-XXX])
      const summaryMatch = header.match(/\[\w+-\d+\]\s+(.+)/);
      return {
        date: dateMatch?.[1] ?? '',
        taskId: taskMatch?.[1] ?? '',
        summary: summaryMatch?.[1] ?? header,
        confidence: 'HIGH',   // Enrichi depuis decisions-graph.json dans getAll()
        scope: 'global',      // Enrichi depuis decisions-graph.json dans getAll()
        raw: block,
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
| 1 | `log()` appende une entrée dans `decisions.log` ET met à jour `decisions-graph.json` | ⬜ |
| 2 | `getRecent(5)` retourne les 5 dernières entrées | ⬜ |
| 3 | `getAll()` retourne toutes les entrées avec les champs `summary`, `date`, `confidence`, `scope`, `raw` | ⬜ |
| 3b | `getAll()` enrichit `confidence` et `scope` depuis `decisions-graph.json` quand disponible | ⬜ |
| 4 | `getRelevant(task)` trouve les entrées liées par mots-clés | ⬜ |
| 5 | `parseEntries` parse correctement le format texte | ⬜ |
| 6 | Retourne `[]` (pas une exception) si `decisions.log` n'existe pas encore | ⬜ |
| 7 | `checkContradictions()` détecte un conflit `CONTRADICTS` actif et retourne les deux IDs | ⬜ |
| 8 | `checkContradictions()` ignore une contradiction si l'une des décisions est `SUPERSEDES` | ⬜ |
| 9 | `log()` retourne `{ id, contradictions }` si contradiction détectée (pour alerte à l'utilisateur) | ⬜ |
| 10 | Tests unitaires couvrent log + graphe + détection de contradictions + getAll | ⬜ |
