# Phase 3 — Tâche 3.2 : CLI.js

## Objectif

Créer l'interface CLI interactive avec `readline` et `chalk`. MVP minimaliste — pas d'Ink pour l'instant. La CLI doit permettre de piloter toutes les phases et commandes Workflow depuis le terminal.

## Dépendances

- Phase 2 ✅

## Fichiers à Créer

- `src/interfaces/CLI.js` [CRÉER]
- `src/index.js` [CRÉER] — point d'entrée

## Commandes CLI

```bash
workflow init "Nom du projet"    # Initialise un projet dans le dossier courant
workflow start                   # Lance/reprend les phases projet
workflow run                     # Exécute la prochaine tâche
workflow status                  # Affiche l'état du projet
workflow watch                   # Mode annotation passive (watcher)
workflow version list            # Liste les versions (Phase 5)
workflow version create v1.5 "Description"
workflow version switch v1.5
workflow version complete
workflow version hotfix v1.0.1 "raison"
```

## Mode Annotation Passive — `workflow watch`

Quand l'utilisateur code manuellement, il n'a pas envie d'être interrompu. `workflow watch` observe les modifications du projet en arrière-plan et pose des questions de clarification sous forme de fichiers dans `.workflow/questions/`.

```
.workflow/questions/
├── 2026-04-03-Q001.md    # En attente de réponse
└── 2026-04-03-Q002.md    # Répondue
```

**Format d'une question** :
```markdown
# Q001 — 2026-04-03

Tu viens de créer `src/cache.js` avec une référence à Redis.

Je l'enregistre comme décision permanente dans `tech-stack.json` ?
Redis sera marqué comme dépendance de la v1.5.

**Réponds en remplaçant `[...]` ci-dessous :**
> [ oui / non / plus-tard ]

**Raison (optionnel) :**
>
```

L'utilisateur répond en éditant le fichier. Au prochain `workflow start` (ou immédiatement si `watch` tourne), Workflow intègre la réponse :
- `oui` → décision enregistrée dans `decisions-graph.json` + `tech-stack.json`
- `non` → question archivée, aucune action
- `plus-tard` → question conservée pour la prochaine session

**Déclencheurs du watcher** (seuil configurable) :
- Nouveau fichier créé dans `src/`
- Import d'une dépendance non listée dans `tech-stack.json`
- Fichier modifié hors des `filesToModify` de la tâche courante

**Pas d'interruption** : le watcher ne bloque jamais le travail en cours. Les questions s'accumulent silencieusement.

## Implémentation — `CLI.js`

```javascript
// src/interfaces/CLI.js
import readline from 'readline';
import chalk from 'chalk';

export class CLI {
  constructor() {
    this.rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout,
    });
  }

  // Afficher un message
  display(message) {
    console.log(message);
  }

  // Afficher un message d'erreur
  error(message) {
    console.error(chalk.red(`❌ ${message}`));
  }

  // Afficher un succès
  success(message) {
    console.log(chalk.green(`✅ ${message}`));
  }

  // Afficher un avertissement
  warn(message) {
    console.log(chalk.yellow(`⚠️  ${message}`));
  }

  // Afficher un header
  header(title) {
    console.log('\n' + chalk.bold.blue(`─── ${title} ───`));
  }

  // Poser une question — retourne la réponse
  ask(question) {
    return new Promise(resolve => {
      this.rl.question(chalk.cyan(`\n${question}\n> `), resolve);
    });
  }

  // Demander une confirmation o/n
  async confirm(question) {
    const answer = await this.ask(`${question} (o/n)`);
    return answer.trim().toLowerCase() === 'o' || answer.trim().toLowerCase() === 'oui';
  }

  // Afficher le statut Workflow
  displayStatus(project, versionCtx) {
    this.header('Workflow Status');
    console.log(`Projet     : ${chalk.bold(project.name)}`);
    console.log(`Phase      : ${chalk.yellow(project.status)}`);
    console.log(`Version    : ${project.currentVersion ? chalk.green(project.currentVersion) : chalk.dim('aucune')}`);

    if (versionCtx) {
      console.log(`Tâches     : ${chalk.green(versionCtx.doneTasks.length)} terminées / ${chalk.yellow(versionCtx.pendingTasks.length)} en attente`);
    }
  }

  // Fermer proprement
  close() {
    this.rl.close();
  }
}
```

## `src/index.js` — Point d'entrée

```javascript
// src/index.js
import { CLI } from './interfaces/CLI.js';
import { WorkflowAgent } from './core/WorkflowAgent.js';

const cli = new CLI();
const agent = new WorkflowAgent(process.cwd(), cli);

const [,, command, ...args] = process.argv;

try {
  switch (command) {
    case 'init':
      await agent.init(args[0] ?? await cli.ask('Nom du projet :'));
      break;
    case 'start':
      await agent.start();
      break;
    case 'run':
      await agent.run();
      break;
    case 'status':
      await agent.status();
      break;
    case 'watch':
      await agent.watch();
      break;
    case 'daemon':
      await agent.daemon();
      break;
    case 'onboard':
      await agent.onboard();
      break;
    case 'doc':
      // workflow doc generate
      if (args[0] === 'generate') await agent.docGenerate?.() ?? cli.display('[Phase 6]');
      else cli.display('Usage: workflow doc generate');
      break;
    case 'audit':
      await agent.audit?.() ?? cli.display('[Phase 6]');
      break;
    case 'estimate':
      await agent.estimate?.(args[0]) ?? cli.display('[Phase 6]');
      break;
    case 'version':
      await agent.version(args);
      break;
    default:
      cli.display('Usage: workflow <init|start|run|status|watch|daemon|onboard|doc|audit|estimate|version>');
  }
} catch (err) {
  cli.error(err.message);
  process.exit(1);
} finally {
  cli.close();
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `cli.ask()` retourne une Promise qui se résout avec la saisie | ⬜ |
| 2 | `cli.confirm()` accepte o/oui et n/non | ⬜ |
| 3 | `workflow --help` affiche la liste des commandes | ⬜ |
| 4 | Les erreurs sont affichées en rouge avec le préfixe ❌ | ⬜ |
| 5 | `cli.close()` ferme proprement readline sans laisser le process suspendu | ⬜ |
| 6 | `workflow watch` démarre sans bloquer et crée un fichier question si nouveau fichier détecté | ⬜ |
| 7 | `workflow start` intègre les réponses aux questions en attente avant de continuer | ⬜ |
| 8 | Une question avec réponse `plus-tard` est conservée pour la session suivante | ⬜ |
