# Phase 4 — Tâche 4.1 : MCPServer.js

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
  error(msg) { process.stderr.write(`[workflow-mcp] ERROR: ${msg}\n`); }
  warn(msg) { process.stderr.write(`[workflow-mcp] WARN: ${msg}\n`); }
  success() {}
  header() {}
  async ask() { return ''; }
  async confirm() { return true; }
  displayStatus() {}
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
        return this.agent.versions.complete();

      case 'workflow_version_hotfix':
        return this.agent.versions.hotfix(args.name, args.reason);

      case 'workflow_version_add_task': {
        const { TaskManager } = await import('../tools/TaskManager.js');
        const tm = new TaskManager(this.projectRoot);
        const taskId = await tm.createTask(args.version, args.task);
        return { success: true, taskId };
      }

      default:
        throw new Error(`Outil inconnu : ${name}`);
    }
  }

  getToolDefinitions() {
    return [
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
        description: 'Complète la version active (merge dans main, bilan)',
        inputSchema: { type: 'object', properties: {} },
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
| 5 | Outil inconnu retourne une erreur propre (pas un crash) | ⬜ |
| 6 | Le serveur est détecté et listé dans Claude Code via MCP | ⬜ |
