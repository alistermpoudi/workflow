# Phase 8 — Tâche 8.3 : TelegramBot.py

## Objectif

Créer `TelegramBot.py` — un bot Telegram qui agit comme **client du protocole MCP**. Permet à l'utilisateur de gérer son projet Workflow depuis n'importe où : briefing matin, prochaine tâche, validation de tâches, approbation des allowed_commands, alerte sur les contradictions de décisions.

> **Pilier load-bearing #6.** Le bot Telegram **n'a aucune logique métier propre** — il appelle MCP. Si on changeait demain de "Telegram" à "Discord" ou "Signal", on garderait 95% du code.

## Dépendances

- Phase 6 ✅ (`MCPServer`)
- Tâche 6.3 ✅ (`AllowedCommandsPolicy`)

## Fichiers à Créer

- `src/workflow/interfaces/telegram_bot.py` [CRÉER]
- `src/workflow/interfaces/mcp_client.py` [CRÉER — client MCP générique réutilisable]
- `tests/unit/test_telegram_bot.py` [CRÉER]

## Architecture — TelegramBot comme client MCP

```
┌─────────────────┐         ┌───────────────┐         ┌──────────────┐
│  Telegram User  │ ──────▶ │  TelegramBot  │ ──────▶ │  MCPServer   │
│   (mobile)      │ ◀────── │  (Python)     │ ◀────── │  (Workflow)  │
└─────────────────┘         └───────────────┘         └──────────────┘
                                   │                          │
                                   └─── stdio MCP protocol ───┘
```

`TelegramBot` ne lit pas `.workflow/` directement — il passe par `MCPClient` qui parle au `MCPServer`.

## Configuration

```yaml
# Dans workflow.config.yaml
telegram:
  bot_token: "${TELEGRAM_BOT_TOKEN}"
  authorized_user_ids: [123456789]   # User ID Telegram de l'utilisateur
  daily_briefing_time: "09:00"       # Heure du briefing matinal
  notify_on_failures: true
  notify_on_contradictions: true
```

## Commandes Disponibles

| Commande Telegram | Outil MCP correspondant |
|-------------------|------------------------|
| `/status` | `workflow_get_project_context()` |
| `/next` | `workflow_get_current_task()` |
| `/done <task>` | `workflow_mark_task_done(task)` |
| `/skip <task>` | `workflow_mark_task_failed(task, reason)` |
| `/decisions` | `workflow_search_decisions(query)` |
| `/skills` | `workflow_list_skills()` |
| `/curate` | `workflow_curate_skills()` |
| `/version switch <v>` | `workflow_version_switch(v)` |
| `/version create <v> <desc>` | `workflow_version_create(v, desc)` |
| `/audit` | `workflow_audit()` |
| `/allow <cmd>` | `workflow_approve_command(cmd, scope='prefix')` |
| `/help` | Aide intégrée |

## Implémentation — `MCPClient` générique

```python
# src/workflow/interfaces/mcp_client.py
import asyncio
import json
from contextlib import AsyncExitStack
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client


class MCPClient:
    """Client MCP réutilisable — utilisé par TelegramBot, RestAPI, futurs clients."""

    def __init__(self, server_command: str, server_args: list[str] | None = None):
        self.server_command = server_command
        self.server_args = server_args or []
        self._exit_stack = AsyncExitStack()
        self._session: ClientSession | None = None

    async def connect(self):
        params = StdioServerParameters(
            command=self.server_command,
            args=self.server_args,
        )
        read, write = await self._exit_stack.enter_async_context(stdio_client(params))
        self._session = await self._exit_stack.enter_async_context(
            ClientSession(read, write)
        )
        await self._session.initialize()

    async def call_tool(self, name: str, arguments: dict | None = None) -> dict:
        if self._session is None:
            raise RuntimeError('MCPClient non connecté — appeler connect() d\'abord')
        result = await self._session.call_tool(name, arguments or {})
        # Extraire le texte du premier content block
        for content in result.content:
            if content.type == 'text':
                try:
                    return json.loads(content.text)
                except json.JSONDecodeError:
                    return {'text': content.text}
        return {}

    async def list_tools(self) -> list[dict]:
        if self._session is None:
            raise RuntimeError('MCPClient non connecté')
        result = await self._session.list_tools()
        return [{'name': t.name, 'description': t.description} for t in result.tools]

    async def close(self):
        await self._exit_stack.aclose()

    async def __aenter__(self):
        await self.connect()
        return self

    async def __aexit__(self, *args):
        await self.close()
```

## Implémentation — `TelegramBot`

```python
# src/workflow/interfaces/telegram_bot.py
import asyncio
import yaml
from pathlib import Path
from telegram import Update
from telegram.ext import (
    Application, CommandHandler, MessageHandler, ContextTypes, filters,
)

from workflow.interfaces.mcp_client import MCPClient


class TelegramBot:
    def __init__(self, project_root: str, config: dict):
        self.project_root = Path(project_root)
        self.config = config['telegram']
        self.token = self.config['bot_token']
        self.authorized_users = set(self.config.get('authorized_user_ids', []))
        self.mcp = MCPClient('workflow-mcp', ['--path', str(self.project_root)])
        self.app: Application | None = None

    def _check_auth(self, user_id: int) -> bool:
        if not self.authorized_users:
            return True  # Tous autorisés (config dev)
        return user_id in self.authorized_users

    async def start(self):
        await self.mcp.connect()
        self.app = Application.builder().token(self.token).build()

        # Handlers
        self.app.add_handler(CommandHandler('status', self.cmd_status))
        self.app.add_handler(CommandHandler('next', self.cmd_next))
        self.app.add_handler(CommandHandler('done', self.cmd_done))
        self.app.add_handler(CommandHandler('skip', self.cmd_skip))
        self.app.add_handler(CommandHandler('decisions', self.cmd_decisions))
        self.app.add_handler(CommandHandler('skills', self.cmd_skills))
        self.app.add_handler(CommandHandler('curate', self.cmd_curate))
        self.app.add_handler(CommandHandler('version', self.cmd_version))
        self.app.add_handler(CommandHandler('audit', self.cmd_audit))
        self.app.add_handler(CommandHandler('allow', self.cmd_allow))
        self.app.add_handler(CommandHandler('help', self.cmd_help))

        # Briefing quotidien
        if self.app.job_queue:
            from datetime import time
            briefing_h, briefing_m = map(
                int, self.config.get('daily_briefing_time', '09:00').split(':')
            )
            self.app.job_queue.run_daily(
                self.send_daily_briefing,
                time=time(hour=briefing_h, minute=briefing_m),
                chat_id=self.authorized_users.copy().pop() if self.authorized_users else None,
            )

        await self.app.initialize()
        await self.app.start()
        await self.app.updater.start_polling()

    async def stop(self):
        if self.app:
            await self.app.updater.stop()
            await self.app.stop()
            await self.app.shutdown()
        await self.mcp.close()

    # ─── Commandes ────────────────────────────────────────────────────

    async def cmd_status(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return await update.message.reply_text('Non autorisé.')
        result = await self.mcp.call_tool('workflow_get_project_context')
        await update.message.reply_text(self._format_status(result))

    async def cmd_next(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return
        result = await self.mcp.call_tool('workflow_get_current_task')
        task = result.get('task', {})
        if not task:
            await update.message.reply_text('Aucune tâche en attente.')
            return
        msg = (
            f"📋 *{task.get('id')}* : {task.get('title')}\n\n"
            f"_Intent_: {task.get('intent', '—')}\n\n"
            f"Critères:\n" + '\n'.join(f"• {c}" for c in task.get('criteria', []))
        )
        await update.message.reply_text(msg, parse_mode='Markdown')

    async def cmd_done(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return
        if not ctx.args:
            return await update.message.reply_text('Usage : /done TASK-XXX')
        task_id = ctx.args[0]
        await self.mcp.call_tool('workflow_mark_task_done', {'task_id': task_id})
        await update.message.reply_text(f'✅ {task_id} marquée comme terminée.')

    async def cmd_skip(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return
        if len(ctx.args) < 2:
            return await update.message.reply_text('Usage : /skip TASK-XXX raison')
        task_id = ctx.args[0]
        reason = ' '.join(ctx.args[1:])
        await self.mcp.call_tool(
            'workflow_mark_task_failed',
            {'task_id': task_id, 'reason': reason},
        )
        await update.message.reply_text(f'⏭ {task_id} reportée : {reason}')

    async def cmd_decisions(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return
        query = ' '.join(ctx.args) if ctx.args else ''
        result = await self.mcp.call_tool(
            'workflow_search_decisions',
            {'query': query},
        )
        decisions = result.get('decisions', [])[:5]
        if not decisions:
            return await update.message.reply_text('Aucune décision trouvée.')
        msg = '\n\n'.join(
            f"📌 {d.get('summary', '')}\n_{d.get('reason', '')}_"
            for d in decisions
        )
        await update.message.reply_text(msg, parse_mode='Markdown')

    async def cmd_skills(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return
        result = await self.mcp.call_tool('workflow_list_skills')
        skills = result.get('skills', [])
        msg = f"🧠 {len(skills)} skills accumulés :\n" + '\n'.join(
            f"• {s.get('name')} ({s.get('usage_count', 0)} usages)"
            for s in skills[:20]
        )
        await update.message.reply_text(msg)

    async def cmd_curate(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return
        await update.message.reply_text('🔄 Curator en cours...')
        result = await self.mcp.call_tool('workflow_curate_skills', {'dry_run': False})
        await update.message.reply_text(
            f"✅ Curator terminé.\n"
            f"Supprimés: {len(result.get('deleted', []))}\n"
            f"Consolidés: {len(result.get('consolidated', []))}\n"
            f"Promus globaux: {len(result.get('promoted', []))}\n"
            f"Archivés: {len(result.get('archived', []))}"
        )

    async def cmd_version(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return
        if not ctx.args:
            result = await self.mcp.call_tool('workflow_version_list')
            versions = result.get('versions', [])
            msg = '\n'.join(
                f"{'⭐' if v.get('active') else '  '} {v['name']} — {v['status']}"
                for v in versions
            )
            return await update.message.reply_text(msg or 'Aucune version.')

        sub = ctx.args[0]
        if sub == 'switch' and len(ctx.args) >= 2:
            result = await self.mcp.call_tool(
                'workflow_version_switch', {'version': ctx.args[1]},
            )
            await update.message.reply_text(
                result.get('message', 'OK')
            )
        elif sub == 'create' and len(ctx.args) >= 3:
            await self.mcp.call_tool(
                'workflow_version_create',
                {'name': ctx.args[1], 'description': ' '.join(ctx.args[2:])},
            )
            await update.message.reply_text(f'Version {ctx.args[1]} créée.')

    async def cmd_audit(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return
        result = await self.mcp.call_tool('workflow_audit')
        await update.message.reply_text(self._format_audit(result))

    async def cmd_allow(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        if not self._check_auth(update.effective_user.id):
            return
        if not ctx.args:
            return await update.message.reply_text('Usage : /allow <commande>')
        command = ' '.join(ctx.args)
        await self.mcp.call_tool(
            'workflow_approve_command',
            {'command': command, 'scope': 'prefix'},
        )
        await update.message.reply_text(f'✅ Approuvée : `{command}`', parse_mode='Markdown')

    async def cmd_help(self, update: Update, ctx: ContextTypes.DEFAULT_TYPE):
        msg = (
            "🤖 *Workflow Bot*\n\n"
            "/status — état du projet\n"
            "/next — prochaine tâche\n"
            "/done TASK-XXX — marquer terminée\n"
            "/skip TASK-XXX raison — reporter\n"
            "/decisions [query] — chercher dans les décisions\n"
            "/skills — lister les skills\n"
            "/curate — lancer le Curator\n"
            "/version [list|switch|create] — gérer les versions\n"
            "/audit — détecter divergences\n"
            "/allow <cmd> — approuver une commande\n"
            "/help — cette aide"
        )
        await update.message.reply_text(msg, parse_mode='Markdown')

    # ─── Briefing quotidien ───────────────────────────────────────────

    async def send_daily_briefing(self, ctx: ContextTypes.DEFAULT_TYPE):
        result = await self.mcp.call_tool('workflow_daily_briefing')
        msg = result.get('text', 'Briefing indisponible.')
        for user_id in self.authorized_users:
            try:
                await ctx.bot.send_message(chat_id=user_id, text=msg, parse_mode='Markdown')
            except Exception:
                continue

    # ─── Formatage ────────────────────────────────────────────────────

    @staticmethod
    def _format_status(result: dict) -> str:
        project = result.get('project', {})
        return (
            f"📂 *{project.get('name', '?')}*\n"
            f"Phase : {project.get('status', '?')}\n"
            f"Version active : {project.get('active_version', 'aucune')}\n"
            f"Tâches done : {result.get('done_count', 0)}\n"
            f"Tâches pending : {result.get('pending_count', 0)}"
        )

    @staticmethod
    def _format_audit(result: dict) -> str:
        issues = result.get('issues', [])
        if not issues:
            return '✅ Aucune divergence détectée.'
        return '\n'.join(f"⚠️ {i}" for i in issues[:10])
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `MCPClient.connect()` initialise une session stdio avec workflow-mcp | ⬜ |
| 2 | `MCPClient.call_tool()` parse le résultat JSON correctement | ⬜ |
| 3 | `MCPClient` supporte le context manager `async with` | ⬜ |
| 4 | `_check_auth()` refuse les user_id non listés dans la config | ⬜ |
| 5 | `/status` appelle `workflow_get_project_context` et formate proprement | ⬜ |
| 6 | `/next` formate la tâche avec intent + critères en Markdown | ⬜ |
| 7 | `/done TASK-001` appelle `workflow_mark_task_done` | ⬜ |
| 8 | `/curate` affiche un récap (deleted, consolidated, promoted, archived) | ⬜ |
| 9 | `/allow <cmd>` ajoute la commande en scope='prefix' | ⬜ |
| 10 | Briefing quotidien envoyé à `daily_briefing_time` | ⬜ |
| 11 | Aucune logique métier dans TelegramBot — uniquement formatage + appels MCP | ⬜ |
| 12 | Tests : mock MCPClient, simuler `/status`, `/next`, `/done` | ⬜ |

## Notes d'Implémentation

- **Multi-user** : la config `authorized_user_ids` est une liste — on peut autoriser plusieurs collaborateurs sur le même projet.
- **Sécurité** : le bot stocke aucun secret côté Telegram. Toutes les opérations passent par MCP local.
- **Réseau** : le bot tourne **localement** (sur la machine du dev ou un serveur perso). Il n'expose pas le projet sur internet — il pull depuis Telegram.
- **Extension Discord/Signal** : remplacer `python-telegram-bot` par le SDK approprié, garder `MCPClient` intact. Code partagé ≥80%.
