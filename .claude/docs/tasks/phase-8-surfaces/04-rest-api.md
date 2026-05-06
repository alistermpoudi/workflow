# Phase 8 — Tâche 8.4 : RestAPI.py

## Objectif

Créer `RestAPI.py` — une **API REST locale** (FastAPI) qui expose Workflow comme service HTTP. Comme `TelegramBot`, c'est un **client du protocole MCP**. Permet : intégrations CI/CD, dashboards web, automation tierce, hooks Git/IDE custom.

> **Pilier load-bearing #6.** Aucune logique métier propre — wrappe MCP. Si on veut demain un GraphQL ou un gRPC, on ré-écrit cette couche fine.

## Dépendances

- Phase 6 ✅ (`MCPServer`)
- Tâche 8.3 ✅ (`MCPClient` réutilisable)

## Fichiers à Créer

- `src/workflow/interfaces/rest_api.py` [CRÉER]
- `tests/integration/test_rest_api.py` [CRÉER]

## Configuration

```yaml
# Dans workflow.config.yaml
rest_api:
  host: "127.0.0.1"          # localhost par défaut — PAS exposé sur internet
  port: 8765
  api_key: "${WORKFLOW_API_KEY}"   # Auth simple par bearer token
  cors_origins: ["http://localhost:3000"]  # pour dashboard web local
```

## Endpoints

```
GET   /health                          → status du serveur
GET   /project                          → workflow_get_project_context
GET   /tasks/current                    → workflow_get_current_task
GET   /tasks?version=v1.0&status=...    → workflow_list_tasks
POST  /tasks/{task_id}/done             → workflow_mark_task_done
POST  /tasks/{task_id}/failed           → workflow_mark_task_failed
GET   /decisions?query=...              → workflow_search_decisions
POST  /decisions                        → workflow_log_decision
GET   /decisions/graph                  → workflow_get_decision_graph
GET   /skills                           → workflow_list_skills
POST  /skills/curate?dry_run=false      → workflow_curate_skills
GET   /versions                         → workflow_version_list
POST  /versions                         → workflow_version_create
POST  /versions/{name}/switch           → workflow_version_switch
POST  /versions/{name}/complete         → workflow_version_complete
POST  /commands/run                     → workflow_run_command (avec policy check)
POST  /commands/approve                 → workflow_approve_command
GET   /audit                            → workflow_audit
GET   /briefing                         → workflow_daily_briefing
GET   /onboard                          → workflow_onboard
```

## Implémentation

```python
# src/workflow/interfaces/rest_api.py
import os
from pathlib import Path
from typing import Annotated
from fastapi import FastAPI, HTTPException, Header, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel

from workflow.interfaces.mcp_client import MCPClient


class WorkflowAPIConfig(BaseModel):
    host: str = '127.0.0.1'
    port: int = 8765
    api_key: str | None = None
    cors_origins: list[str] = []


def create_app(project_root: str, config: WorkflowAPIConfig) -> FastAPI:
    app = FastAPI(title='Workflow REST API', version='1.0.0')
    mcp = MCPClient('workflow-mcp', ['--path', str(project_root)])

    # CORS pour dashboards web locaux
    if config.cors_origins:
        app.add_middleware(
            CORSMiddleware,
            allow_origins=config.cors_origins,
            allow_credentials=True,
            allow_methods=['*'],
            allow_headers=['*'],
        )

    async def auth(authorization: Annotated[str | None, Header()] = None):
        """Auth bearer token (optionnel — si api_key configurée)"""
        if not config.api_key:
            return  # Pas d'auth requise
        if not authorization or not authorization.startswith('Bearer '):
            raise HTTPException(401, 'Bearer token requis')
        token = authorization.removeprefix('Bearer ').strip()
        if token != config.api_key:
            raise HTTPException(403, 'Token invalide')

    @app.on_event('startup')
    async def startup():
        await mcp.connect()

    @app.on_event('shutdown')
    async def shutdown():
        await mcp.close()

    # ─── Health & Project ─────────────────────────────────────────────

    @app.get('/health')
    async def health():
        return {'status': 'ok', 'project_root': str(project_root)}

    @app.get('/project', dependencies=[Depends(auth)])
    async def get_project():
        return await mcp.call_tool('workflow_get_project_context')

    # ─── Tasks ────────────────────────────────────────────────────────

    @app.get('/tasks/current', dependencies=[Depends(auth)])
    async def get_current_task():
        return await mcp.call_tool('workflow_get_current_task')

    @app.get('/tasks', dependencies=[Depends(auth)])
    async def list_tasks(version: str | None = None, status: str | None = None):
        return await mcp.call_tool(
            'workflow_list_tasks',
            {'version': version, 'status': status},
        )

    @app.post('/tasks/{task_id}/done', dependencies=[Depends(auth)])
    async def mark_task_done(task_id: str):
        return await mcp.call_tool('workflow_mark_task_done', {'task_id': task_id})

    class FailedBody(BaseModel):
        reason: str

    @app.post('/tasks/{task_id}/failed', dependencies=[Depends(auth)])
    async def mark_task_failed(task_id: str, body: FailedBody):
        return await mcp.call_tool(
            'workflow_mark_task_failed',
            {'task_id': task_id, 'reason': body.reason},
        )

    # ─── Decisions ────────────────────────────────────────────────────

    @app.get('/decisions', dependencies=[Depends(auth)])
    async def search_decisions(query: str = ''):
        return await mcp.call_tool('workflow_search_decisions', {'query': query})

    class DecisionBody(BaseModel):
        task_id: str
        decision: str
        reason: str
        scope: str = 'global'

    @app.post('/decisions', dependencies=[Depends(auth)])
    async def log_decision(body: DecisionBody):
        return await mcp.call_tool('workflow_log_decision', body.model_dump())

    @app.get('/decisions/graph', dependencies=[Depends(auth)])
    async def get_decision_graph():
        return await mcp.call_tool('workflow_get_decision_graph')

    # ─── Skills ───────────────────────────────────────────────────────

    @app.get('/skills', dependencies=[Depends(auth)])
    async def list_skills():
        return await mcp.call_tool('workflow_list_skills')

    @app.post('/skills/curate', dependencies=[Depends(auth)])
    async def curate_skills(dry_run: bool = False):
        return await mcp.call_tool('workflow_curate_skills', {'dry_run': dry_run})

    # ─── Versions ─────────────────────────────────────────────────────

    @app.get('/versions', dependencies=[Depends(auth)])
    async def list_versions():
        return await mcp.call_tool('workflow_version_list')

    class VersionBody(BaseModel):
        name: str
        description: str

    @app.post('/versions', dependencies=[Depends(auth)])
    async def create_version(body: VersionBody):
        return await mcp.call_tool('workflow_version_create', body.model_dump())

    @app.post('/versions/{name}/switch', dependencies=[Depends(auth)])
    async def switch_version(name: str):
        result = await mcp.call_tool('workflow_version_switch', {'version': name})
        if result.get('blocked'):
            raise HTTPException(409, result.get('reason', 'Repo non propre'))
        return result

    @app.post('/versions/{name}/complete', dependencies=[Depends(auth)])
    async def complete_version(name: str):
        return await mcp.call_tool('workflow_version_complete', {'version': name})

    # ─── Commands & Policy ────────────────────────────────────────────

    class CommandBody(BaseModel):
        command: str

    @app.post('/commands/run', dependencies=[Depends(auth)])
    async def run_command(body: CommandBody):
        result = await mcp.call_tool('workflow_run_command', {'command': body.command})
        if not result.get('authorized'):
            raise HTTPException(403, result.get('error', 'Non autorisée'))
        return result

    class ApproveBody(BaseModel):
        command: str
        scope: str = 'prefix'

    @app.post('/commands/approve', dependencies=[Depends(auth)])
    async def approve_command(body: ApproveBody):
        return await mcp.call_tool('workflow_approve_command', body.model_dump())

    # ─── Audit / Briefing / Onboard ───────────────────────────────────

    @app.get('/audit', dependencies=[Depends(auth)])
    async def audit():
        return await mcp.call_tool('workflow_audit')

    @app.get('/briefing', dependencies=[Depends(auth)])
    async def briefing():
        return await mcp.call_tool('workflow_daily_briefing')

    @app.get('/onboard', dependencies=[Depends(auth)])
    async def onboard():
        return await mcp.call_tool('workflow_onboard')

    return app


def run_server(project_root: str, config_path: Path | None = None):
    """Point d'entrée pour `workflow rest-api start`"""
    import yaml
    import uvicorn

    config_data = {}
    if config_path and config_path.exists():
        with open(config_path) as f:
            full = yaml.safe_load(f) or {}
        config_data = full.get('rest_api', {})

    # Résoudre les variables d'environnement
    api_key = config_data.get('api_key', '')
    if isinstance(api_key, str) and api_key.startswith('${'):
        api_key = os.environ.get(api_key[2:-1], '')
        config_data['api_key'] = api_key

    config = WorkflowAPIConfig(**config_data)
    app = create_app(project_root, config)
    uvicorn.run(app, host=config.host, port=config.port)
```

## Sécurité

- **Localhost par défaut** (`127.0.0.1`) — l'API n'est pas exposée sur internet sans config explicite
- **Bearer token** simple via header `Authorization: Bearer <key>` (suffisant pour usage local)
- **Pas de CORS wildcard** — les origines sont explicitement listées
- **Pas d'opération sur d'autres projets** — `project_root` est figé au démarrage

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `GET /health` retourne 200 sans auth | ⬜ |
| 2 | `GET /project` retourne 401 sans Bearer si api_key configurée | ⬜ |
| 3 | `GET /project` retourne le contexte projet avec auth correcte | ⬜ |
| 4 | `POST /tasks/TASK-001/done` appelle `workflow_mark_task_done` | ⬜ |
| 5 | `POST /versions/v2.0/switch` retourne 409 si repo non propre | ⬜ |
| 6 | `POST /commands/run` avec commande non autorisée retourne 403 | ⬜ |
| 7 | CORS ne laisse passer que les origines configurées | ⬜ |
| 8 | Aucune logique métier dans rest_api.py — uniquement appels MCP | ⬜ |
| 9 | `MCPClient` est connecté au startup et fermé au shutdown | ⬜ |
| 10 | Tests d'intégration : 6 endpoints clés (mock MCP) | ⬜ |

## Notes d'Implémentation

- **WebSocket** : non couvert ici. Phase ultérieure pour streaming des logs `ExecutionLoop`. À ajouter si besoin réel apparaît.
- **Rate limiting** : pas nécessaire pour usage local mono-utilisateur. Si on déploie sur LAN multi-user, ajouter `slowapi`.
- **Deprecation** : utiliser `lifespan` de FastAPI au lieu de `on_event` (deprecated en 0.110+). Implémentation laissée volontairement simple pour le MVP.
- **OpenAPI** : auto-généré par FastAPI (`/docs`). Permet à un dashboard web tiers de générer son client en TypeScript.
