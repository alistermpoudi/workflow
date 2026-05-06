# Phase 5 — Tâche 5.3 : InFlowCorrection (Ctrl+T pendant l'agent)

## Objectif

Permettre au dev d'**interrompre l'agent en cours d'exécution** pour le corriger immédiatement, sans perdre le contexte de la session. Le dev appuie sur Ctrl+T (ou utilise `/correct` côté MCP), tape sa correction, et :

1. La correction devient un skill `USER_OVERRIDE` dans le context approprié
2. La tâche en cours est **retentée** avec la nouvelle instruction injectée
3. La session continue, l'agent ne perd ni la tâche ni le contexte

> **Pilier load-bearing #2 (4ème source d'apprentissage).** C'est la voie la plus haute en signal — le dev voit l'agent dériver et le corrige instantanément, en plein contexte. Plus précieux que le `workflow teach` froid car le contexte du moment est riche (tâche, fichiers, dernière sortie LLM).

## Dépendances

- Phase 1-4 ✅
- Tâche 3.1 ✅ (`ExecutionLoop`)
- Tâche 3.4 ✅ (`TeachSystem` pour créer le skill USER_OVERRIDE)
- Tâche 5.2 ✅ (`WorkflowAgent`)

## Fichiers à Créer

- `src/workflow/core/in_flow_corrector.py` [CRÉER]
- Modification : `src/workflow/tools/execution_loop.py` (hook d'interruption)
- Modification : `src/workflow/interfaces/cli.py` (handler Ctrl+T)
- Modification : `src/workflow/interfaces/mcp_server.py` (`workflow_correct` tool)

## UX en CLI

```
$ workflow run

▶ TASK-027 : Endpoint /users
  Génération du code...

  ### MODIFIER src/api/routes.py
  <<<<<<< SEARCH
  from fastapi import APIRouter
  ...
  =======
  from fastapi import APIRouter, FastAPI
  router = APIRouter()
  router.middleware("auth")(authenticate)  ← Workflow génère ça

[CTRL+T pressé par le dev]

✋ Correction en flow
   Que dois-je corriger ?
   > pas @middleware, utilise FastAPI Depends pattern : Depends(get_current_user)

✓ Skill USER_OVERRIDE créé dans backend.fastapi : "auth-via-depends-not-middleware"
↻ Retry de TASK-027 avec la correction injectée...

  ### MODIFIER src/api/routes.py
  ...
  router.get("/users")(get_users, dependencies=[Depends(get_current_user)])  ← corrigé

✓ TASK-027 validée en 2 tentatives.
```

## UX en MCP (Claude Code, Cursor)

L'agent hôte appelle `workflow_correct(task_id, correction)` au lieu d'attendre la fin de l'exécution :

```
Workflow MCP — Génération du code...
[résultat partiel]

Claude Code : Tu utilises @middleware mais on est sur FastAPI. Je corrige.
→ workflow_correct(task_id="TASK-027", correction="utiliser Depends(get_current_user) au lieu de @middleware")
→ Skill créé. Retry en cours.

[résultat final]
```

## Implémentation

### `InFlowCorrector` — orchestrateur

```python
# src/workflow/core/in_flow_corrector.py
import asyncio
from dataclasses import dataclass, field
from datetime import datetime, timezone

from workflow.core.teach_system import TeachSystem
from workflow.core.context_registry import ContextRegistry
from workflow.core.skill_manager import SkillManager


@dataclass
class CorrectionRequest:
    task_id: str
    correction_text: str
    last_llm_output: str = ''
    timestamp: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())


@dataclass
class CorrectionResult:
    skill_name: str
    context: str
    retry_triggered: bool


class InFlowCorrector:
    """
    Capture une correction de l'utilisateur en plein milieu d'une exécution
    et la transforme en skill USER_OVERRIDE + signal de retry.
    """

    def __init__(self, teach_system: TeachSystem, registry: ContextRegistry, io):
        self.teach = teach_system
        self.registry = registry
        self.io = io
        self._pending_correction: CorrectionRequest | None = None
        self._interrupt_event = asyncio.Event()

    # ─── Côté CLI : capture Ctrl+T ────────────────────────────────────

    def signal_interrupt(self):
        """Appelé par le handler Ctrl+T du CLI"""
        self._interrupt_event.set()

    async def is_interrupt_pending(self) -> bool:
        return self._interrupt_event.is_set()

    async def consume_interrupt(self, task_ctx: dict, last_llm_output: str) -> CorrectionResult | None:
        """
        Consommer un Ctrl+T : prompter l'utilisateur, créer le skill, retourner pour retry.
        Retourne None si l'utilisateur abandonne.
        """
        self._interrupt_event.clear()

        self.io.print('\n✋ [bold yellow]Correction en flow[/bold yellow]')
        self.io.print('   Que dois-je corriger ?')
        correction_text = self.io.prompt('>', default='').strip()
        if not correction_text:
            self.io.print_warning('Annulé. Reprise sans correction.')
            return None

        return await self.apply_correction(
            CorrectionRequest(
                task_id=task_ctx['task']['id'],
                correction_text=correction_text,
                last_llm_output=last_llm_output[-1500:],
            )
        )

    # ─── Côté MCP : appel direct ──────────────────────────────────────

    async def apply_correction(self, request: CorrectionRequest) -> CorrectionResult:
        """Créer un skill USER_OVERRIDE depuis la correction et signaler un retry"""
        # Inférer le context : le plus spécifique des active_contexts du projet courant
        active_contexts = self._get_active_contexts()
        target_context = active_contexts[-1] if active_contexts else '_global'

        # Créer le skill
        skill_result = self.teach.teach(
            rule=request.correction_text,
            context=target_context,
            tags=['in_flow_correction', f'task:{request.task_id}'],
            visibility='shared',
            title=f'inflow-{request.task_id.lower()}',
        )

        self.io.print_success(
            f'✓ Skill USER_OVERRIDE créé dans {target_context} : '
            f'{skill_result["name"]}'
        )

        return CorrectionResult(
            skill_name=skill_result['name'],
            context=target_context,
            retry_triggered=True,
        )

    @staticmethod
    def _get_active_contexts() -> list[str]:
        """Lire les active_contexts du projet courant"""
        from pathlib import Path
        import json
        project_json = Path.cwd() / '.workflow' / 'project.json'
        if project_json.exists():
            try:
                data = json.loads(project_json.read_text())
                return data.get('active_contexts', ['_global'])
            except json.JSONDecodeError:
                pass
        return ['_global']
```

### Hook dans `ExecutionLoop` (Phase 3 — modification)

```python
# src/workflow/tools/execution_loop.py — ajout

class ExecutionLoop:
    def __init__(self, ..., in_flow_corrector: InFlowCorrector | None = None):
        ...
        self.corrector = in_flow_corrector

    async def run(self, task_ctx: dict) -> dict:
        ...
        for attempt in range(MAX_RETRIES + 1):
            ...
            raw_response = await self.llm.ask(prompt, role='code_generation')

            # Vérifier si une interruption a été signalée pendant la génération
            if self.corrector and await self.corrector.is_interrupt_pending():
                result = await self.corrector.consume_interrupt(task_ctx, raw_response)
                if result and result.retry_triggered:
                    self.io.print_info('↻ Retry avec la correction injectée...')
                    continue   # Re-prompt avec le nouveau skill maintenant injecté
            ...
```

Le skill `USER_OVERRIDE` est lu par `LLMContextLoader` au prochain `get_task_context()`, donc le retry suivant inclura la correction automatiquement.

### CLI handler (Phase 5 — modification)

```python
# src/workflow/interfaces/cli.py — ajout
import signal
import asyncio

def install_correction_handler(corrector: InFlowCorrector):
    """SIGQUIT (Ctrl+\\) ou keypress Ctrl+T selon plateforme"""
    # Sur macOS/Linux : SIGINFO (Ctrl+T) ou un keystroke listener
    # Pour la portabilité, on utilise SIGINT (Ctrl+C) avec confirmation,
    # ou on ajoute un listener sur stdin si TTY interactif

    def handler(signum, frame):
        corrector.signal_interrupt()

    # Sur darwin uniquement, SIGINFO existe
    import sys
    if sys.platform == 'darwin':
        signal.signal(signal.SIGINFO, handler)
    else:
        # Linux/Windows : utiliser un fallback (signal séparé ou keystroke)
        signal.signal(signal.SIGUSR1, handler)
```

> **Note plateforme** : sur macOS, Ctrl+T génère SIGINFO nativement. Sur Linux, il faut un workaround (signal SIGUSR1 ou listener stdin). À documenter dans la CLI.

### Tool MCP `workflow_correct`

```python
# Dans MCPServer
types.Tool(
    name='workflow_correct',
    description=(
        'Corriger l\'agent en plein milieu d\'une tâche. '
        'Crée un skill USER_OVERRIDE dans le context actif et signale un retry.'
    ),
    inputSchema={
        'type': 'object',
        'properties': {
            'task_id': {'type': 'string'},
            'correction': {'type': 'string', 'minLength': 5},
            'context': {'type': 'string', 'description': 'Optionnel — sinon active context le plus spécifique'},
        },
        'required': ['task_id', 'correction'],
    },
)

@server.call_tool()
async def workflow_correct(args: dict):
    request = CorrectionRequest(
        task_id=args['task_id'],
        correction_text=args['correction'],
    )
    result = await corrector.apply_correction(request)
    return {'skill_name': result.skill_name, 'context': result.context}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `signal_interrupt()` met `_interrupt_event` à True (asyncio safe) | ⬜ |
| 2 | `is_interrupt_pending()` reflète l'état | ⬜ |
| 3 | `consume_interrupt()` prompte l'utilisateur, crée le skill, clear l'event | ⬜ |
| 4 | Si l'utilisateur tape vide, retourne None et l'agent continue sans retry | ⬜ |
| 5 | `apply_correction()` crée un skill USER_OVERRIDE avec tag `in_flow_correction` | ⬜ |
| 6 | Le skill est créé dans le context le plus spécifique des active_contexts | ⬜ |
| 7 | Hors projet, le skill va dans `_global` | ⬜ |
| 8 | `ExecutionLoop` consume l'interruption AVANT le retry suivant | ⬜ |
| 9 | Le retry inclut le nouveau skill via `LLMContextLoader.get_task_context()` | ⬜ |
| 10 | MCP tool `workflow_correct` appelle directement `apply_correction()` | ⬜ |
| 11 | Sur macOS, SIGINFO trigge `signal_interrupt()` | ⬜ |
| 12 | Tests : capture interrupt, apply correction, no-correction abort, MCP path | ⬜ |

## Notes d'Implémentation

- **Race condition** : `is_interrupt_pending()` doit être checké au bon moment (entre génération et application des fichiers, idéalement avant). Sinon le dev peut voir le code être appliqué avant que sa correction ne soit prise en compte.
- **Stabilité du keystroke** : SIGINFO macOS fonctionne. Linux : SIGUSR1 + un wrapper script qui mappe Ctrl+T → SIGUSR1 via `stty`. Documenter dans la CLI.
- **MCP path est plus robuste** : le client (Claude Code) appelle `workflow_correct` directement quand il détecte que l'agent dérive, sans dépendre d'un signal OS.
- **Historique des corrections** : enregistrer un audit log dans `~/.workflow/corrections.log` avec horodatage + tâche + correction. Permet de retrouver le contexte si on veut affiner le skill plus tard.
- **Pas de retry infini** : si après 3 corrections la tâche échoue toujours, escalade comme l'ExecutionLoop normal.
