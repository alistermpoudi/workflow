# Phase 4 — Tâche 4.1 : MCPServer.js

> **Note** : Ce dossier s'appelle `phase-4-versioning` pour des raisons historiques.
> Il correspond à la **Phase 4 — MCP Server (Workflow Core)** dans CLAUDE.md.

## Objectif

Exposer tous les outils Workflow via le protocole MCP. En mode Workflow Core, Workflow n'est pas le LLM — il fournit les outils. L'agent hôte (Claude Code, Cursor…) réfléchit et appelle les outils. La règle de sécurité `allowed_commands` s'applique à toutes les exécutions.

## Dépendances

- Phase 3 ✅

## Fichiers à Créer

- `src/interfaces/MCPServer.js` [CRÉER]
- `bin/workflow-mcp.js` [CRÉER] — point d'entrée MCP

## Implémentation

```javascript
// src/interfaces/MCPServer.js
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { CallToolRequestSchema, ListToolsRequestSchema } from '@modelcontextprotocol/sdk/types.js';
import { WorkflowAgent } from '../core/WorkflowAgent.js';
import { ProjectMemory } from '../core/ProjectMemory.js';

// I/O silencieuse pour le mode MCP (pas d'affichage CLI)
class SilentIO {
  display() {}
  error(msg) { throw new Error(msg); } // Les erreurs doivent remonter
  warn(msg) { /* no-op — les warnings sont perdus en mode MCP */ }
  success() {}
  header() {}
  displayStatus() {}

  // En mode MCP, les confirmations doivent être EXPLICITEMENT passées via les paramètres de l'outil
  // SilentIO refuse toute confirmation implicite pour éviter les actions silencieuses
  async confirm(question) {
    throw new Error(
      `Confirmation requise mais non fournie en mode MCP : "${question}"\n` +
      `Passe le paramètre "force: true" à l'outil MCP pour confirmer explicitement.`
    );
  }

  async ask(question) {
    throw new Error(`Input requis en mode MCP : "${question}". Utilise les paramètres de l'outil.`);
  }
}

export class MCPServer {
  constructor(projectRoot) {
    this.projectRoot = projectRoot;
    this.agent = new WorkflowAgent(projectRoot, new SilentIO());
    this.memory = new ProjectMemory(projectRoot);

    this.server = new Server(
      { name: 'workflow', version: '0.1.0' },
      { capabilities: { tools: {} } }
    );

    this.registerHandlers();
  }

  registerHandlers() {
    this.server.setRequestHandler(ListToolsRequestSchema, () => ({
      tools: this.getToolDefinitions(),
    }));

    this.server.setRequestHandler(CallToolRequestSchema, async (req) => {
      const { name, arguments: args } = req.params;
      try {
        const result = await this.callTool(name, args ?? {});
        return {
          content: [{ type: 'text', text: JSON.stringify(result, null, 2) }],
        };
      } catch (err) {
        return {
          content: [{ type: 'text', text: `Error: ${err.message}` }],
          isError: true,
        };
      }
    });
  }

  async callTool(name, args) {
    switch (name) {
      // ── Phase Projet ──────────────────────────────────────────────
      case 'workflow_start_project':
        return this.agent.init(args.name);

      case 'workflow_save_discovery':
        return this.agent.phases.saveDiscovery(args.answers);

      case 'workflow_propose_features':
        return this.agent.phases.proposeFeatures();

      case 'workflow_save_features':
        return this.agent.phases.saveFeatures(args.validated);

      case 'workflow_generate_tasks':
        return this.agent.phases.generateTasks();

      case 'workflow_validate_task':
        return this.agent.phases.validateTask(args.taskId, args.approved);

      case 'workflow_set_tech_stack':
        return this.agent.phases.setTechStack(args.stack);

      case 'workflow_get_project_context':
        return this.memory.getProjectSummary();

      case 'workflow_get_current_task': {
        const version = await this.memory.getActiveVersion();
        if (!version) return { error: 'Aucune version active' };
        const { TaskManager } = await import('../tools/TaskManager.js');
        const tm = new TaskManager(this.projectRoot);
        return tm.getNextTask(version);
      }

      case 'workflow_mark_task_done': {
        const version = await this.memory.getActiveVersion();
        const { TaskManager } = await import('../tools/TaskManager.js');
        const tm = new TaskManager(this.projectRoot);
        await tm.markDone(version, args.taskId);
        return { success: true, taskId: args.taskId };
      }

      case 'workflow_mark_task_failed': {
        const version = await this.memory.getActiveVersion();
        const { TaskManager } = await import('../tools/TaskManager.js');
        const tm = new TaskManager(this.projectRoot);
        await tm.deferTask(version, args.taskId, 'next', args.reason);
        return { success: true };
      }

      case 'workflow_log_decision': {
        const { DecisionsLog } = await import('../core/DecisionsLog.js');
        const log = new DecisionsLog(this.projectRoot);
        await log.log(args.taskId, args.decision, args.reason);
        return { success: true };
      }

      case 'workflow_search_codebase': {
        // Phase 6 — CodeIndexer non encore disponible
        return { error: 'workflow_search_codebase disponible en Phase 6 (CodeIndexer)' };
      }

      // ── Gestion des Versions ──────────────────────────────────────
      case 'workflow_version_list':
        return this.agent.versions.list();

      case 'workflow_version_create':
        return this.agent.versions.create(args.name, args.description);

      case 'workflow_version_switch':
        return this.agent.versions.switch(args.version);

      case 'workflow_version_complete':
        // Si force: true, SilentIO ne sera pas invoqué pour la confirmation
        if (args.force) {
          return this.agent.versions.complete({ skipConfirmation: true });
        }
        return this.agent.versions.complete();

      case 'workflow_version_hotfix':
        return this.agent.versions.hotfix(args.name, args.reason);

      case 'workflow_version_add_task': {
        const { TaskManager } = await import('../tools/TaskManager.js');
        const tm = new TaskManager(this.projectRoot);
        const taskId = await tm.createTask(args.version, args.task);
        return { success: true, taskId };
      }

      // ── Documentation & Utilitaires ──────────────────────────────
      case 'workflow_doc_generate':
        // Phase 7 — génère README + ARCHITECTURE.md + CHANGELOG depuis .workflow/
        return this.agent.docGenerate?.() ?? { message: 'Disponible en Phase 7 (Génération & Audit)' };

      case 'workflow_audit':
        // Phase 7 — détecte les divergences entre code et tâches
        return this.agent.audit?.() ?? { message: 'Disponible en Phase 7 (Génération & Audit)' };

      case 'workflow_estimate': {
        // Phase 7 — estimations basées sur l'historique git réel
        const version = args.version ?? await this.memory.getActiveVersion();
        return this.agent.estimate?.(version) ?? { message: 'Disponible en Phase 7 (Génération & Audit)' };
      }

      case 'workflow_onboard':
        // Phase 5 — onboarding nouveau dev en 30 secondes depuis .workflow/
        return this.agent.onboard?.() ?? { message: 'Disponible en Phase 5 (Présence & Intégrations)' };

      default:
        throw new Error(`Outil inconnu : ${name}`);
    }
  }

  getToolDefinitions() {
    return [
      {
        name: 'workflow_start_project',
        description: 'Initialise un nouveau projet Workflow dans le répertoire courant',
        inputSchema: {
          type: 'object',
          properties: {
            name: { type: 'string', description: 'Nom du projet' },
            description: { type: 'string', description: 'Description courte du projet' },
          },
          required: ['name'],
        },
      },
      {
        name: 'workflow_save_discovery',
        description: 'Sauvegarde les réponses de la phase Discovery et génère vision.md',
        inputSchema: {
          type: 'object',
          properties: {
            answers: { type: 'string', description: 'Réponses de l\'utilisateur aux questions de découverte' },
          },
          required: ['answers'],
        },
      },
      {
        name: 'workflow_propose_features',
        description: 'Génère une proposition de fonctionnalités structurée par version depuis vision.md',
        inputSchema: { type: 'object', properties: {} },
      },
      {
        name: 'workflow_save_features',
        description: 'Sauvegarde les fonctionnalités validées dans features.json',
        inputSchema: {
          type: 'object',
          properties: {
            validated: { type: 'object', description: 'Objet features validé (structure: { "v1.0": [...], "v1.5": [...] })' },
          },
          required: ['validated'],
        },
      },
      {
        name: 'workflow_generate_tasks',
        description: 'Génère les fichiers de tâches TASK-XXX.md pour toutes les versions depuis features.json',
        inputSchema: { type: 'object', properties: {} },
      },
      {
        name: 'workflow_validate_task',
        description: 'Valide ou rejette une tâche générée avant qu\'elle soit enregistrée',
        inputSchema: {
          type: 'object',
          properties: {
            taskId: { type: 'string', description: 'Ex: TASK-003' },
            approved: { type: 'boolean', description: 'true = valider, false = rejeter' },
          },
          required: ['taskId', 'approved'],
        },
      },
      {
        name: 'workflow_set_tech_stack',
        description: 'Définit et sauvegarde la stack technique dans tech-stack.json',
        inputSchema: {
          type: 'object',
          properties: {
            stack: { type: 'object', description: 'Objet stack technique (language, framework, database, build_validate, test, allowed_commands)' },
          },
          required: ['stack'],
        },
      },
      {
        name: 'workflow_get_project_context',
        description: 'Retourne le contexte complet du projet courant (nom, stack, version active, tâche en cours)',
        inputSchema: { type: 'object', properties: {} },
      },
      {
        name: 'workflow_get_current_task',
        description: 'Retourne la prochaine tâche à exécuter (contenu complet du fichier TASK-XXX.md)',
        inputSchema: { type: 'object', properties: {} },
      },
      {
        name: 'workflow_mark_task_done',
        description: 'Marque une tâche comme terminée et met à jour progress.json',
        inputSchema: {
          type: 'object',
          properties: { taskId: { type: 'string', description: 'Ex: TASK-003' } },
          required: ['taskId'],
        },
      },
      {
        name: 'workflow_mark_task_failed',
        description: 'Marque une tâche comme reportée avec une raison',
        inputSchema: {
          type: 'object',
          properties: {
            taskId: { type: 'string' },
            reason: { type: 'string' },
          },
          required: ['taskId', 'reason'],
        },
      },
      {
        name: 'workflow_log_decision',
        description: 'Enregistre une décision technique dans decisions.log',
        inputSchema: {
          type: 'object',
          properties: {
            taskId: { type: 'string' },
            decision: { type: 'string', description: 'La décision prise (ex: ORM: Prisma)' },
            reason: { type: 'string', description: 'Justification de la décision' },
          },
          required: ['taskId', 'decision', 'reason'],
        },
      },
      {
        name: 'workflow_version_list',
        description: 'Liste toutes les versions du projet avec leur statut',
        inputSchema: { type: 'object', properties: {} },
      },
      {
        name: 'workflow_version_create',
        description: 'Crée une nouvelle version et une branche Git workflow/vX.Y',
        inputSchema: {
          type: 'object',
          properties: {
            name: { type: 'string', description: 'Ex: v1.5' },
            description: { type: 'string' },
          },
          required: ['name'],
        },
      },
      {
        name: 'workflow_version_switch',
        description: 'Switche vers une version (bloque si repo non propre)',
        inputSchema: {
          type: 'object',
          properties: { version: { type: 'string' } },
          required: ['version'],
        },
      },
      {
        name: 'workflow_version_complete',
        description: 'Marque la version active comme COMPLETED et merge sur la branche par défaut',
        inputSchema: {
          type: 'object',
          properties: {
            force: {
              type: 'boolean',
              description: 'Si true, complète même avec des tâches en attente (confirmation explicite)',
            },
          },
        },
      },
      {
        name: 'workflow_version_hotfix',
        description: 'Crée un hotfix sur une version précédente',
        inputSchema: {
          type: 'object',
          properties: {
            name: { type: 'string', description: 'Ex: v1.0.1' },
            reason: { type: 'string' },
          },
          required: ['name', 'reason'],
        },
      },
      {
        name: 'workflow_version_add_task',
        description: 'Ajoute une tâche à une version existante',
        inputSchema: {
          type: 'object',
          properties: {
            version: { type: 'string' },
            task: { type: 'object', description: 'Objet tâche complet' },
          },
          required: ['version', 'task'],
        },
      },
      {
        name: 'workflow_doc_generate',
        description: 'Génère README + ARCHITECTURE.md + CHANGELOG depuis .workflow/ (Phase 7)',
        inputSchema: { type: 'object', properties: {} },
      },
      {
        name: 'workflow_audit',
        description: 'Détecte les divergences entre le code source et les tâches (Phase 7)',
        inputSchema: { type: 'object', properties: {} },
      },
      {
        name: 'workflow_estimate',
        description: 'Estimations de temps basées sur l\'historique git réel (Phase 7)',
        inputSchema: {
          type: 'object',
          properties: { version: { type: 'string', description: 'Version à estimer (ex: v1.5). Défaut: version active.' } },
        },
      },
      {
        name: 'workflow_onboard',
        description: 'Génère un résumé d\'onboarding pour un nouveau développeur depuis .workflow/ (Phase 5)',
        inputSchema: { type: 'object', properties: {} },
      },
    ];
  }

  async start() {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    process.stderr.write('[workflow-mcp] Serveur MCP démarré\n');
  }
}
```

## `bin/workflow-mcp.js`

```javascript
#!/usr/bin/env node
// bin/workflow-mcp.js
import { MCPServer } from '../src/interfaces/MCPServer.js';

const server = new MCPServer(process.cwd());
await server.start();
```

## Configuration dans Claude Code

```json
// ~/.claude/claude_desktop_config.json (ou claude.json selon version)
{
  "mcpServers": {
    "workflow": {
      "command": "node",
      "args": ["/chemin/vers/workflow/bin/workflow-mcp.js"],
      "cwd": "/chemin/vers/ton/projet"
    }
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `workflow_get_project_context` retourne les infos projet sans erreur | ⬜ |
| 2 | `workflow_get_current_task` retourne le contenu complet de la tâche | ⬜ |
| 3 | `workflow_mark_task_done` met à jour `progress.json` | ⬜ |
| 4 | `workflow_log_decision` appende dans `decisions.log` | ⬜ |
| 5 | `workflow_doc_generate`, `workflow_audit`, `workflow_estimate` retournent un message clair "Disponible en Phase 7" | ⬜ |
| 5b | `workflow_onboard` retourne un message clair "Disponible en Phase 5" | ⬜ |
| 6 | Outil inconnu retourne une erreur propre (pas un crash) | ⬜ |
| 7 | Le serveur est détecté et listé dans Claude Code via MCP | ⬜ |
| 8 | `workflow_start_project` est visible par les clients MCP (dans `getToolDefinitions()`) | ⬜ |
| 9 | Les 6 outils Phase 2 (`workflow_save_discovery`, `workflow_propose_features`, `workflow_save_features`, `workflow_generate_tasks`, `workflow_validate_task`, `workflow_set_tech_stack`) sont disponibles dans `callTool()` et `getToolDefinitions()` | ⬜ |
| 10 | `workflow_version_complete` sans `force` throw si des tâches sont en attente | ⬜ |
| 11 | `workflow_version_complete` avec `force: true` complète sans demander confirmation | ⬜ |
