# Phase 4 — Tâche 4.1 : MCPServer.py

## Objectif

Créer le serveur MCP Python qui expose tous les outils Workflow. Utilise le **SDK Python officiel `mcp`** avec transport `stdio`. Les outils MCP sont de simples wrappers sur les modules Core déjà construits.

> **Sécurité** : `MCPServer` ne peut exécuter que les commandes listées dans `tech-stack.json#allowed_commands`.
> Toute autre commande est rejetée avec une erreur explicite.

## Dépendances

- Phases 1, 2, 3 complètes ✅

## Fichiers à Créer

- `src/workflow/interfaces/mcp_server.py` [CRÉER]

## Configuration Claude Code / Cursor

```json
// ~/.claude/claude_desktop_config.json  (ou .cursor/mcp.json)
{
  "mcpServers": {
    "workflow": {
      "command": "workflow-mcp",
      "args": ["--path", "/chemin/vers/projet"]
    }
  }
}
```

## Outils MCP Exposés

```
── Phase Projet ──────────────────────────────────────────
workflow_start_project(description)
workflow_save_discovery(answers)
workflow_propose_features()
workflow_save_features(validated_json)
workflow_generate_tasks()
workflow_validate_task(task_id, approved)
workflow_set_tech_stack(stack_json)

── Gestion des Versions ──────────────────────────────────
workflow_version_list()
workflow_version_create(name, description)
workflow_version_switch(version)          ← bloque si repo non propre
workflow_version_add_task(version, task_json)
workflow_version_complete()

── Contexte & Exécution ──────────────────────────────────
workflow_get_current_task()
workflow_get_project_context()
workflow_search_codebase(query)
workflow_mark_task_done(task_id)
workflow_mark_task_failed(task_id, reason)
workflow_log_decision(task_id, decision, reason)
workflow_run_command(command)             ← whitelist only
```

## Implémentation

```python
# src/workflow/interfaces/mcp_server.py
import asyncio
import json
from pathlib import Path

import typer
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp import types

from workflow.core.project_memory import ProjectMemory
from workflow.core.context_manager import ContextManager
from workflow.core.decisions_log import DecisionsLog
from workflow.tools.task_manager import TaskManager
from workflow.tools.version_manager import VersionManager
from workflow.llm.llm_provider import LLMProvider

app_cli = typer.Typer()


def create_server(project_root: str) -> Server:
    server = Server('workflow')
    memory = ProjectMemory(project_root)
    tasks = TaskManager(project_root)
    decisions = DecisionsLog(project_root)
    llm = LLMProvider.from_config_file()
    ctx_manager = ContextManager(project_root, llm)
    version_mgr = VersionManager(project_root)

    @server.list_tools()
    async def list_tools() -> list[types.Tool]:
        return [
            types.Tool(
                name='workflow_get_current_task',
                description='Retourner la tâche courante avec son contexte complet (fichiers, décisions, skills).',
                inputSchema={'type': 'object', 'properties': {}, 'required': []},
            ),
            types.Tool(
                name='workflow_get_project_context',
                description='Retourner le contexte système du projet (project.json + tech-stack.json).',
                inputSchema={'type': 'object', 'properties': {}, 'required': []},
            ),
            types.Tool(
                name='workflow_mark_task_done',
                description='Marquer une tâche comme terminée dans progress.json.',
                inputSchema={
                    'type': 'object',
                    'properties': {'task_id': {'type': 'string'}},
                    'required': ['task_id'],
                },
            ),
            types.Tool(
                name='workflow_mark_task_failed',
                description='Marquer une tâche comme échouée.',
                inputSchema={
                    'type': 'object',
                    'properties': {
                        'task_id': {'type': 'string'},
                        'reason': {'type': 'string'},
                    },
                    'required': ['task_id', 'reason'],
                },
            ),
            types.Tool(
                name='workflow_log_decision',
                description='Enregistrer une décision technique dans DecisionsLog.',
                inputSchema={
                    'type': 'object',
                    'properties': {
                        'task_id': {'type': 'string'},
                        'decision': {'type': 'string'},
                        'reason': {'type': 'string'},
                    },
                    'required': ['task_id', 'decision', 'reason'],
                },
            ),
            types.Tool(
                name='workflow_version_list',
                description='Lister toutes les versions du projet.',
                inputSchema={'type': 'object', 'properties': {}, 'required': []},
            ),
            types.Tool(
                name='workflow_version_switch',
                description='Changer la version active. Bloque si le repo git n\'est pas propre.',
                inputSchema={
                    'type': 'object',
                    'properties': {'version': {'type': 'string'}},
                    'required': ['version'],
                },
            ),
            types.Tool(
                name='workflow_run_command',
                description='Exécuter une commande de la whitelist allowed_commands.',
                inputSchema={
                    'type': 'object',
                    'properties': {'command': {'type': 'string'}},
                    'required': ['command'],
                },
            ),
        ]

    @server.call_tool()
    async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
        try:
            result = await _dispatch(
                name, arguments, memory, tasks, decisions, ctx_manager, version_mgr, project_root
            )
            return [types.TextContent(type='text', text=json.dumps(result, ensure_ascii=False, indent=2))]
        except Exception as e:
            return [types.TextContent(type='text', text=json.dumps({'error': str(e)}))]

    return server


async def _dispatch(
    name: str,
    args: dict,
    memory: ProjectMemory,
    tasks: TaskManager,
    decisions: DecisionsLog,
    ctx: ContextManager,
    version_mgr: VersionManager,
    project_root: str,
) -> dict:
    if name == 'workflow_get_project_context':
        return await ctx.get_system_context()

    elif name == 'workflow_get_current_task':
        project = await memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            return {'error': 'Aucune version active.'}
        progress = await memory.get_progress(version)
        pending = (progress or {}).get('pending', [])
        if not pending:
            return {'message': 'Toutes les tâches sont terminées.'}
        task_id = pending[0]
        return await ctx.get_task_context(version, task_id)

    elif name == 'workflow_mark_task_done':
        project = await memory.get_project()
        version = (project or {}).get('active_version')
        await tasks.mark_done(version, args['task_id'])
        ctx.invalidate_cache()
        return {'ok': True, 'task_id': args['task_id']}

    elif name == 'workflow_mark_task_failed':
        project = await memory.get_project()
        version = (project or {}).get('active_version')
        await tasks.mark_failed(version, args['task_id'], args['reason'])
        ctx.invalidate_cache()
        return {'ok': True, 'task_id': args['task_id']}

    elif name == 'workflow_log_decision':
        from datetime import datetime, timezone
        await decisions.add({
            'task_id': args['task_id'],
            'summary': args['decision'],
            'reason': args['reason'],
            'confidence': 'HIGH',
            'scope': 'task',
            'date': datetime.now(timezone.utc).date().isoformat(),
        })
        return {'ok': True}

    elif name == 'workflow_version_list':
        return {'versions': await version_mgr.list_versions()}

    elif name == 'workflow_version_switch':
        result = await version_mgr.switch(args['version'])
        if result.get('blocked'):
            return {'error': result['reason']}
        ctx.invalidate_cache()
        return {'ok': True, 'version': args['version']}

    elif name == 'workflow_run_command':
        tech_stack = await memory.get_tech_stack() or {}
        allowed = tech_stack.get('allowed_commands', [])
        command = args['command']
        if not any(command.startswith(a.split()[0]) for a in allowed):
            return {'error': f'Commande non autorisée : {command!r}'}
        import asyncio as aio
        proc = await aio.create_subprocess_shell(
            command,
            stdout=aio.subprocess.PIPE,
            stderr=aio.subprocess.STDOUT,
            cwd=project_root,
        )
        stdout, _ = await proc.communicate()
        return {
            'exit_code': proc.returncode,
            'output': stdout.decode('utf-8', errors='replace'),
        }

    return {'error': f'Outil inconnu : {name}'}


@app_cli.command()
def main(
    path: str = typer.Option('.', '--path', '-p', help='Dossier du projet cible'),
) -> None:
    """Démarrer le serveur MCP Workflow (transport stdio)."""
    project_root = str(Path(path).resolve())
    server = create_server(project_root)

    async def run() -> None:
        async with stdio_server() as (read_stream, write_stream):
            await server.run(read_stream, write_stream, server.create_initialization_options())

    asyncio.run(run())
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `workflow-mcp --path .` démarre sans erreur | ⬜ |
| 2 | `workflow_get_current_task` retourne le contexte complet (task + fichiers + décisions + skills) | ⬜ |
| 3 | `workflow_mark_task_done` appelle `tasks.mark_done()` et `ctx.invalidate_cache()` | ⬜ |
| 4 | `workflow_run_command` vérifie la whitelist avant d'exécuter | ⬜ |
| 5 | `workflow_run_command` rejette une commande non listée avec un message explicite | ⬜ |
| 6 | `workflow_version_switch` retourne `{'error': ...}` si repo non propre | ⬜ |
| 7 | `workflow_log_decision` persiste dans `DecisionsLog` | ⬜ |
| 8 | Toutes les erreurs sont catchées et retournées en `{'error': str}` (pas de crash serveur) | ⬜ |
