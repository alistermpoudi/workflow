# Phase 3 — Tâche 3.3 : WorkflowAgent.py

## Objectif

Créer `WorkflowAgent.py` — l'orchestrateur principal du mode Agent autonome. Il câble ensemble `PhaseManager`, `ExecutionPhase`, `ExecutionLoop`, `SyncChecker`, et `LLMProvider`. C'est le point d'entrée de toutes les commandes CLI.

## Dépendances

- Phases 1 et 2 complètes ✅
- Tâches 3.1 ✅, 3.2 ✅

## Fichiers à Créer

- `src/workflow/core/workflow_agent.py` [CRÉER]

## Implémentation

```python
# src/workflow/core/workflow_agent.py
import json
from pathlib import Path

from workflow.core.phase_manager import PhaseManager
from workflow.core.project_memory import ProjectMemory
from workflow.core.sync_checker import SyncChecker
from workflow.llm.llm_provider import LLMProvider
from workflow.tools.execution_loop import ExecutionLoop


class WorkflowAgent:
    def __init__(self, project_root: str, io, llm: LLMProvider | None = None):
        self.project_root = str(Path(project_root).resolve())
        self.io = io
        self.llm = llm or LLMProvider.from_config_file()
        self.memory = ProjectMemory(self.project_root)
        self.sync = SyncChecker(self.project_root)
        self._phase_manager: PhaseManager | None = None

    def _get_phase_manager(self) -> PhaseManager:
        if self._phase_manager is None:
            self._phase_manager = PhaseManager(self.project_root, self.llm, self.io)
            # Injecter ExecutionLoop dans ExecutionPhase
            tech_stack = {}  # Sera rechargé à l'exécution
            loop = ExecutionLoop(self.project_root, self.llm, self.io, tech_stack)
            self._phase_manager.phases['ACTIVE'].set_execution_loop(loop)
        return self._phase_manager

    async def init(self) -> None:
        """Initialiser un nouveau projet .workflow/"""
        self.io.print_header('Workflow — Initialisation')

        existing = await self.memory.get_project()
        if existing:
            overwrite = self.io.confirm('Un projet .workflow/ existe déjà. Réinitialiser ?')
            if not overwrite:
                return

        name = self.io.prompt('Nom du projet')
        description = self.io.prompt('Description courte (optionnel)', default='')

        await self.memory.init_project({'name': name, 'description': description})
        self.io.print_success(f'Projet "{name}" initialisé dans {self.project_root}/.workflow/')
        self.io.print_info('Lance `workflow run` pour démarrer la phase Discovery.')

    async def run_phase(self) -> None:
        """Exécuter la phase courante"""
        # Vérification de cohérence au démarrage
        sync_result = await self.sync.check()
        if not sync_result.get('ok'):
            for issue in sync_result.get('issues', []):
                self.io.print_warning(issue)
            if not self.io.confirm('Continuer malgré ces avertissements ?'):
                return

        pm = self._get_phase_manager()

        # Recharger tech_stack pour ExecutionLoop si en phase ACTIVE
        phase = await pm.get_current_phase()
        if phase == 'ACTIVE':
            tech_stack = await self.memory.get_tech_stack() or {}
            loop = ExecutionLoop(self.project_root, self.llm, self.io, tech_stack)
            pm.phases['ACTIVE'].set_execution_loop(loop)

        result = await pm.run_current_phase()

        if result.get('completed'):
            new_phase = await pm.get_current_phase()
            self.io.print_success(f'Phase terminée. Phase suivante : {new_phase}')

    async def run_task(self, task_id: str) -> None:
        """Exécuter une tâche spécifique"""
        project = await self.memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            self.io.print_error('Aucune version active.')
            return

        tech_stack = await self.memory.get_tech_stack() or {}
        from workflow.core.context_manager import ContextManager
        ctx_manager = ContextManager(self.project_root, self.llm)

        try:
            task_ctx = await ctx_manager.get_task_context(version, task_id)
        except ValueError as e:
            self.io.print_error(str(e))
            return

        loop = ExecutionLoop(self.project_root, self.llm, self.io, tech_stack)
        result = await loop.run(task_ctx)

        if result.get('success'):
            from workflow.tools.task_manager import TaskManager
            tasks = TaskManager(self.project_root)
            await tasks.mark_done(version, task_id)
            self.io.print_success(f'{task_id} terminée.')
        else:
            self.io.print_error(f'{task_id} échouée : {result.get("error", "")}')

    async def show_status(self) -> None:
        """Afficher l'état courant du projet"""
        project = await self.memory.get_project()
        if not project:
            self.io.print_warning('Aucun projet .workflow/ trouvé. Lance `workflow init`.')
            return

        pm = self._get_phase_manager()
        phase = await pm.get_current_phase()

        self.io.print_header(f'{project.get("name", "Projet")} — État')
        self.io.print(f'  Phase courante : [bold]{phase}[/bold]')
        self.io.print(f'  Version active : [bold]{project.get("active_version", "aucune")}[/bold]')

        version = project.get('active_version')
        if version:
            progress = await self.memory.get_progress(version)
            if progress:
                done = progress.get('done', [])
                pending = progress.get('pending', [])
                failed = progress.get('failed', [])
                self.io.print(f'  Tâches done    : {len(done)}')
                self.io.print(f'  Tâches pending : {len(pending)}')
                if failed:
                    self.io.print(f'  Tâches failed  : {len(failed)} [red]{", ".join(failed)}[/red]')

    async def onboard(self) -> None:
        """Générer un résumé d'onboarding pour un nouveau développeur"""
        project = await self.memory.get_project()
        if not project:
            self.io.print_error('Aucun projet .workflow/ trouvé.')
            return

        vision = await self.memory.get_vision()
        tech_stack = await self.memory.get_tech_stack()
        features = await self.memory.get_features()

        self.io.print_header(f'Onboarding — {project.get("name", "Projet")}')

        onboard_prompt = f"""Tu es Workflow. Génère un onboarding concis pour un nouveau développeur.
Projet : {json.dumps(project, ensure_ascii=False)}
Stack : {json.dumps(tech_stack or {}, ensure_ascii=False)}
Vision (extrait) : {(vision or '')[:500]}
Fonctionnalités : {json.dumps(list((features or {}).keys()), ensure_ascii=False)}

Génère :
1. Résumé du projet (2-3 phrases)
2. Stack expliquée avec raisons
3. État d'avancement
4. Les 3 décisions clés à connaître
5. Première tâche suggérée

Format Markdown, concis."""

        async def on_chunk(chunk: str) -> None:
            self.io.print(chunk, end='')

        await self.llm.stream(onboard_prompt, role='reasoning', on_chunk=on_chunk)
        self.io.print('')
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `init()` crée `project.json` avec name et description | ⬜ |
| 2 | `run_phase()` appelle `SyncChecker.check()` et bloque si problème non confirmé | ⬜ |
| 3 | `run_phase()` recharge `tech_stack` avant d'injecter `ExecutionLoop` en phase ACTIVE | ⬜ |
| 4 | `run_task(task_id)` charge `get_task_context()` et délègue à `ExecutionLoop` | ⬜ |
| 5 | `show_status()` affiche phase, version active, compteurs done/pending/failed | ⬜ |
| 6 | `onboard()` utilise `LLMProvider.stream()` avec `role='reasoning'` | ⬜ |
| 7 | `LLMProvider.from_config_file()` est utilisé par défaut si aucun llm injecté | ⬜ |
| 8 | `_get_phase_manager()` injecte `ExecutionLoop` dans `phases['ACTIVE']` | ⬜ |
