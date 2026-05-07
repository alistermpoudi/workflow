# Phase 0 — Architecture Technique

## Vue d'Ensemble

Workflow est une application **Python 3.12+** structurée en couches clairement séparées. Le principe fondamental : **le core est indépendant de l'interface**. La même logique (`ProjectMemory`, `DecisionsLog`, `ExecutionLoop`) fonctionne en CLI, en MCP, en Telegram ou en API REST. Les interfaces non-Python (VS Code = TypeScript obligatoire) sont des **clients du protocole MCP**, pas des duplications du core.

> Voir aussi : [`02-pillars.md`](02-pillars.md) — détail des 6 bets architecturaux load-bearing.

## Les 7 Piliers Architecturaux

Tout le découpage de phases ci-dessous découle de ces 7 piliers :

| # | Pilier | Phase de build |
|---|--------|----------------|
| 1 | `.workflow/` comme **protocole versionné** (pas dossier ad-hoc) | Phase 1 |
| 2 | **Skills + Curator + 4 sources** (auto_retry, teach, in-flow, ingestion) | Phases 1 + 3 + 5 + 9 |
| 3 | **Multi-LLM par rôle** (`reasoning`, `code_generation`, `fast`, `curator`, `compression`) | Phase 2 |
| 4 | **Décisions actives + graphe rétro-propageant** (CONTRADICTS, SUPERSEDES…) | Phase 1 |
| 5 | **CodePatcher chirurgical** dès le départ (jamais de fichier complet régénéré) | Phase 2 |
| 6 | **MCP Server** comme surface primaire (CLI/VS Code/Telegram = clients du protocole) | Phase 6 |
| 7 | **Contexts spécialisés** — hiérarchie + héritage + templates bundled | Phase 1 |

---

## Architecture en Couches

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CLIENTS DU PROTOCOLE MCP                                                │
│  CLI (typer+rich)  │  VS Code ext (TS)  │  Telegram bot  │  REST API    │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ stdio MCP
┌──────────────────────────────▼───────────────────────────────────────────┐
│  ORCHESTRATEUR                                                           │
│  WorkflowAgent (mode Agent autonome) / MCPServer (mode Workflow Core)   │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────────┐
│  PHASES (revisitables, pas waterfall)                                    │
│  Discovery → Specification → Design → Architecture → Validation → Exec  │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────────┐
│  CORE — Cerveau cognitif                                                 │
│  ProjectMemory   │  LLMContextLoader (tier + compression Hermes-style)  │
│  PhaseManager    │  DecisionsLog + DecisionsGraph (rétro-propagation)   │
│  SkillManager    │  SkillCurator   │  TeachSystem  │  ContextRegistry   │
│  SyncChecker     │  TaskManager    │  InFlowCorrector  │  ProjectIngester│
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────────┐
│  OUTILS                                                                  │
│  FileSystem  │  GitManager  │  CodePatcher (chirurgical, dès Phase 2)   │
│  ExecutionLoop  │  CodeIndexer  │  ParallelExecutor (worktrees)         │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────────┐
│  LLM (multi-modèles par rôle via LiteLLM)                                │
│  reasoning │ code_generation │ fast │ curator │ compression              │
│  PromptBuilder (contexte projet + skills + decisions injectés)           │
└──────────────────────────────────────────────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
        Claude API        DeepSeek API        Ollama (local)
```

---

## Structure des Fichiers Source

```
src/workflow/
├── core/                            # Cerveau cognitif — indépendant de l'interface
│   ├── workflow_agent.py            # Orchestrateur mode Agent (Phase 5)
│   ├── project_memory.py            # Reference impl du protocole .workflow/
│   ├── phase_manager.py             # Orchestre les 5 phases (revisitables)
│   ├── context_manager.py           # Hiérarchie + compression Hermes-style + scoring
│   ├── decisions_log.py             # decisions.log + index SQLite FTS5
│   ├── decisions_graph.py           # Graphe relations + détection contradictions + rétro-propagation
│   ├── skill_manager.py             # Skills cross-projet (CRUD + recherche)
│   ├── skill_curator.py             # Consolidation périodique via LLM role='curator'
│   ├── sync_checker.py              # State drift + branche Git + préconditions
│   └── daemon_heartbeat.py          # Surveillance continue + briefings (Phase 7)
│
├── phases/                          # Les 5 phases projet (revisitables)
│   ├── discovery_phase.py           # Questions → vision.md
│   ├── specification_phase.py       # Propositions → features.json
│   ├── design_system_phase.py       # Style → design.json + screen-flow.md
│   ├── architecture_phase.py        # Stack → tech-stack.json + TASK-001/002
│   ├── validation_phase.py          # Tâches → versions/ (avec mockups)
│   └── execution_phase.py           # Sélection tâche prête → délègue à ExecutionLoop
│
├── tools/                           # Outils techniques
│   ├── filesystem.py                # WorkflowPaths + opérations async
│   ├── git_manager.py               # asyncio.create_subprocess_exec
│   ├── task_manager.py              # CRUD TASK-XXX.md + progress.json
│   ├── version_manager.py           # Cycle de vie versions, pilote GitManager
│   ├── code_patcher.py              # Diffs chirurgicaux + tree-sitter (Phase 2 — pas v1.5)
│   ├── execution_loop.py            # generate_patch → apply → validate → retry → skill
│   ├── code_indexer.py              # Index + recherche ripgrep (Phase 9)
│   ├── parallel_executor.py         # Git worktrees pour tâches indépendantes (Phase 7)
│   ├── watch_mode.py                # watchfiles → .workflow/questions/ (Phase 7)
│   ├── conflict_resolver.py         # Conflits décisions entre devs (Phase 8)
│   └── workflow_library.py          # Patterns cross-projet (Phase 9)
│
├── interfaces/                      # Couches d'interaction (clients du protocole MCP)
│   ├── cli.py                       # typer + rich + RichIO (Phase 5)
│   ├── mcp_server.py                # SDK Python officiel (Phase 6) — surface primaire
│   ├── allowed_commands_policy.py   # Whitelist + apprentissage (Phase 6)
│   ├── github_integration.py        # PR mergée → tâche DONE (Phase 8)
│   ├── rest_api.py                  # FastAPI local (Phase 8)
│   ├── telegram_bot.py              # Client MCP wrapper (Phase 8)
│   └── vscode_extension/            # TypeScript — VS Code API (Phase 8)
│
└── llm/                             # Abstraction LLM
    ├── llm_provider.py              # LiteLLM multi-modèles par rôle
    └── prompt_builder.py            # Prompts avec contexte projet + skills + decisions injectés
```

---

## Structure `.workflow/` Générée dans les Projets Cibles

```
.workflow/
├── project.json
│   # { schema_version, name, description, created_at, status,
│   #   active_version, active_contexts: [_global, mobile, mobile.flutter] }
│
├── vision.md
│   # Sortie Discovery — description libre de l'application
│
├── features.json
│   # Sortie Specification — fonctionnalités par version
│   # { "v1.0": [{ id, name, description, priority, intent }], "v1.5": [...] }
│
├── tech-stack.json
│   # Sortie Architecture — stack + commandes
│   # { language, framework, database, build_validate, test, allowed_commands[] }
│
├── design.json
│   # Sortie DesignSystem — style + couleurs + références
│
├── code-index.json
│   # Mis à jour en continu (Phase 9 — CodeIndexer)
│   # { "src/auth.py": [{ name: "login", type: "function", line: 42 }] }
│
├── decisions.log
│   # Journal actif texte brut — une entrée par décision technique
│   # [date] [TASK-XXX] Décision : ... / Raison : ...
│
├── decisions-graph.json
│   # Relations entre décisions : CONTRADICTS, DEPENDS_ON, SUPERSEDES, REFINES
│   # Niveaux de confiance : USER_OVERRIDE | HIGH | MEDIUM | LOW
│   # Détection automatique de contradictions
│
├── allowed-commands.json
│   # Whitelist + apprentissage — voir Pilier 6
│   # { allowed: [{ command, scope, approved_at, approved_by }], denied: [...] }
│
├── skills/
│   # Skills strictement projet (le reste vit dans les contexts globaux)
│   └── *.md
│
├── questions/
│   # WatchMode — questions posées à l'utilisateur sans interruption
│   └── YYYY-MM-DD-Q001.md
│
├── briefings/
│   # DaemonHeartbeat — briefings quotidiens générés automatiquement
│   └── YYYY-MM-DD.md
│
└── versions/
    ├── v1.0/
    │   ├── meta.json         # { title, description, status, branch, created_at }
    │   ├── progress.json     # { done: [], pending: [], failed: [], deferred: [] }
    │   └── tasks/
    │       ├── TASK-001.md   # Setup + linter (systématique)
    │       ├── TASK-002.md   # Tests + smoke test (systématique)
    │       └── TASK-003.md
    ├── v1.0.1/               # Hotfix
    │   ├── meta.json         # { type: "HOTFIX", parent: "v1.0", ... }
    │   ├── progress.json
    │   └── tasks/
    ├── v1.5/
    └── v2.0/
```

> Voir aussi `~/.workflow/contexts/` (Pilier 7) — l'arborescence globale des contexts hiérarchiques (`_global`, `mobile`, `mobile.flutter`, etc.) qui contient les skills + decisions cross-projet.

---

## `LLMContextLoader` — Hiérarchie de Chargement

C'est le composant le plus critique pour ne pas reproduire le problème du context overflow. Il charge les skills + decisions des **contexts actifs** (Pilier 7) avec scoring pondéré par spécificité.

```python
# Ordre de chargement strict — ne jamais déroger

class LLMContextLoader:
    # Niveau 1 : Système — toujours chargé (~500 tokens max)
    async def get_system_context(self) -> dict:
        return {
            'project': await self.memory.get_project_summary(),
            'tech_stack': await self.memory.get_tech_stack(),
            'active_contexts': await self.memory.get_active_contexts(),
        }

    # Niveau 2 : Version active — chargé au switch de version
    async def get_version_context(self, version: str) -> dict:
        return {
            'meta': await self.memory.get_version_meta(version),
            'progress': await self.memory.get_progress(version),  # IDs only, pas le contenu
        }

    # Niveau 3 : Tâche courante — chargé au start task
    async def get_task_context(self, version: str, task_id: str) -> dict:
        task = await self.tasks.get_task(version, task_id)
        relevant_files = await self.fs.read_selective(task.get('files_to_modify') or [])

        # Skills + avoidances depuis les contexts actifs (Pilier 7)
        active_contexts = await self.memory.get_active_contexts()
        skill_context = self.skills.get_skills_for_context(task, active_contexts)
        avoidances = self.teach.get_active_avoidances(active_contexts)

        # Scoring de pertinence : Score = similarité × récence × scope × confiance
        # USER_OVERRIDE > HIGH > MEDIUM > LOW
        # Budget tokens = 2000 | Seuil = 0.4
        scored_decisions = await self._load_scored_decisions(task)

        return {
            'task': task,
            'relevant_files': relevant_files,
            'relevant_decisions': scored_decisions,
            'skill_context': skill_context,
            'avoidances': avoidances,
        }

    # Niveau 4 : On-demand — uniquement si nécessaire
    async def load_on_demand(self, query: str) -> list[dict]:
        return await self.indexer.query(query)  # CodeIndexer Phase 9
```

**Règle** : Charger uniquement le niveau nécessaire pour l'action en cours. Ne jamais passer de niveau 4 en "contexte de base".

**Scoring de pertinence** : `LLMContextLoader` ne charge pas les décisions mécaniquement. Il calcule `Score = similarité × récence × scope × confiance` et n'inclut que les décisions avec score ≥ 0.4, dans la limite d'un budget de 2000 tokens. Le champ `intent` de la tâche est inclus dans le texte de référence pour le scoring. Skills `USER_OVERRIDE` battent toujours les autres au scoring final.

---

## `SyncChecker` — Détection du State Drift

Au démarrage de chaque session :

```javascript
class SyncChecker:
    async def check(self) -> dict:
        # 0. Valider la conformité au protocole .workflow/ (Pilier 1)
        errors = self.validator.validate_workflow_dir(self.workflow_root)
        if errors:
            return {'type': 'PROTOCOL_INVALID', 'errors': errors}

        # 1. Vérifier cohérence branche Git / version active
        active_branch = await self.git.current_branch()
        active_version = await self.memory.get_active_version()

        if active_branch != f'workflow/{active_version}':
            return {
                'type': 'BRANCH_MISMATCH',
                'message': f'Branche "{active_branch}" ≠ version active "{active_version}"',
            }

        # 2. Détecter fichiers modifiés manuellement depuis la dernière session
        last_session = await self.memory.get_last_session_timestamp()
        modified_files = await self.git.get_modified_since(last_session)

        if modified_files:
            # 3. Soumettre le diff au LLM pour analyse sémantique
            diff = await self.git.get_diff(modified_files)
            return {'type': 'MANUAL_CHANGES', 'files': modified_files, 'diff': diff}

        # 4. Alerter si contradictions actives dans le decisions-graph
        contradictions = await self.graph.find_active_contradictions()
        if contradictions:
            return {'type': 'CONTRADICTIONS', 'count': len(contradictions)}

        return {'type': 'CLEAN'}
```

---

## `ExecutionLoop` — Boucle d'Auto-Correction

```python
class ExecutionLoop:
    async def run(self, task_ctx: dict) -> dict:
        task = task_ctx['task']
        relevant_files = task_ctx['relevant_files']
        relevant_decisions = task_ctx['relevant_decisions']
        skill_context = task_ctx.get('skill_context', '')

        last_error = None
        attempts: list[str] = []

        for attempt in range(MAX_RETRIES + 1):
            # Chercher un fix connu (skill) AVANT de générer
            known_fix = self.skills.find_fix_for_error(last_error['output']) if last_error else None

            # Construire le prompt — patches via CodePatcher (jamais de fichier complet)
            if attempt == 0:
                prompt = PromptBuilder.generate_code(
                    task, relevant_files, relevant_decisions, skill_context
                )
            else:
                prompt = PromptBuilder.generate_code_retry(
                    task, relevant_files, relevant_decisions, last_error,
                    known_fix, skill_context,
                )

            raw_response = await self.llm.ask(prompt, role='code_generation')
            attempts.append(raw_response)

            # Vérifier si l'utilisateur a interrompu (Ctrl+T) — Phase 5
            if self.corrector and await self.corrector.is_interrupt_pending():
                await self.corrector.consume_interrupt(task_ctx, raw_response)
                continue  # Retry avec le nouveau skill USER_OVERRIDE injecté

            # Extraire et persister les décisions techniques annotées
            await self._extract_decisions(task['id'], raw_response)

            # Appliquer les patches via CodePatcher (Pilier 5)
            patches = self.code_patcher.parse_patches(raw_response)
            results = await self.code_patcher.apply(patches)
            failed = [r for r in results if not r.success]
            if failed:
                last_error = {'output': '\n'.join(f'{r.file_path}: {r.error}' for r in failed)}
                if attempt < MAX_RETRIES:
                    continue
                return {'success': False, 'error': last_error['output'], 'attempts': attempts}

            # Valider build + tests
            for cmd in (self.tech_stack['build_validate'], self.tech_stack['test']):
                result = await self._run_command(cmd)
                if result['exit_code'] != 0:
                    last_error = result
                    break
            else:
                # Tout OK
                if attempt > 0 and last_error:
                    # Retry réussi → créer un skill auto (source: auto_retry)
                    await self._maybe_create_skill(task, attempts, last_error)
                await self.tasks.mark_done(task['id'])
                return {'success': True, 'attempts': attempts}

            if attempt >= MAX_RETRIES:
                return {'success': False, 'error': last_error['output'], 'attempts': attempts}
```

---

## `MCPServer` — Outils Exposés avec Sécurité

```python
# Validation stricte via AllowedCommandsPolicy (Phase 6 — apprentissage + 3 niveaux)
async def workflow_run_command(args: dict):
    if not await policy.authorize(args['command']):
        raise PermissionError(f'Commande non autorisée : {args["command"]}')
    ...

# Outils MCP exposés (liste complète Phase 6)
TOOLS = [
    # Phase Projet
    'workflow_start_project',
    'workflow_save_discovery',
    'workflow_propose_features',
    'workflow_save_features',
    'workflow_generate_tasks',
    'workflow_validate_task',
    'workflow_set_tech_stack',

    # Versions
    'workflow_version_list',
    'workflow_version_create',
    'workflow_version_switch',     # bloque si repo non propre
    'workflow_version_add_task',
    'workflow_version_hotfix',     # bloque si repo non propre
    'workflow_version_complete',

    # Contexte & Exécution
    'workflow_get_current_task',
    'workflow_get_project_context',
    'workflow_search_codebase',
    'workflow_mark_task_done',
    'workflow_mark_task_failed',
    'workflow_log_decision',
    'workflow_get_decision_graph',
    'workflow_run_command',         # avec policy check
    'workflow_approve_command',
    'workflow_correct',             # in-flow correction (Phase 5)

    # Apprentissage (Pilier 2 — 4 sources)
    'workflow_teach',
    'workflow_avoid',
    'workflow_learn_from',          # ingestion projet (Phase 9)
    'workflow_curate_skills',       # déclencher Curator
    'workflow_list_skills',

    # Contexts (Pilier 7)
    'workflow_context_list',
    'workflow_context_list_available',
    'workflow_context_create_from_template',
    'workflow_context_create_custom',
    'workflow_context_activate',
    'workflow_context_export',
    'workflow_context_install',
    'workflow_context_fork',
    'workflow_context_promote_skill',

    # Polish
    'workflow_audit',
    'workflow_doc_generate',
    'workflow_estimate',
    'workflow_onboard',
    'workflow_daily_briefing',
]
```

---

## Stack Technique Détaillée

| Composant | Choix | Notes |
|-----------|-------|-------|
| Runtime | Python | 3.12+ |
| Package manager | `uv` | Lockfile + gestion Python version |
| LLM (multi-rôle) | `litellm` | Routage par rôle — Claude, DeepSeek, Ollama |
| Modèle reasoning | `claude-opus-4-7` | Discovery, Specification, Architecture |
| Modèle code | `deepseek-coder-v2` | ExecutionLoop — best HumanEval |
| Modèle fast | `claude-haiku-4-5` | Scoring décisions, préconditions |
| Modèle curator | `claude-sonnet-4-6` | Consolidation skills cross-projet |
| MCP | `mcp` (SDK Python officiel Anthropic) | Transport stdio |
| CLI | `typer` + `rich` (RichIO) | Couleurs, panels, prompts |
| Base de données | `aiosqlite` + SQLite FTS5 | DecisionsLog avec recherche plein-texte |
| Fichiers async | `aiofiles` + `pathlib` | Opérations non-bloquantes |
| AST (CodePatcher) | `tree-sitter` Python + grammaires | **Dès Phase 2** — pas v1.5 |
| Recherche code | `ripgrep` (subprocess) | Index + recherche fast |
| Watch mode | `watchfiles` | Surveillance fichiers (équivalent chokidar) |
| Git | `asyncio.create_subprocess_exec` | GitManager complet |
| YAML | `pyyaml` | Skills frontmatter, config |
| Tests | `pytest` + `pytest-asyncio` | — |
| Linter | `ruff` | Lint + format |
| Type check | `mypy` strict | — |
| VS Code extension | TypeScript (VS Code API) | Client MCP — Phase 8 |
| Telegram | Bot Telegram API (Python) | Client MCP — Phase 8 |
| GitHub | PyGithub ou octokit-py | Client MCP — Phase 8 |

> Justification du choix Python : écosystème IA mature (LiteLLM, tree-sitter, ML), asyncio solide, MCP SDK officiel Python disponible. Les surfaces non-Python (VS Code = TS obligatoire) seront des **clients du protocole MCP** — pas du code partagé.

---

## Roadmap Technique (par seuil de capacité, pas par version produit)

### Seuil 1 — Foundations (Phases 1-3)
Le cerveau cognitif fonctionne. `LLMProvider` route par rôle, `LLMContextLoader` charge en hiérarchie + compresse + résout les contexts actifs, `CodePatcher` applique des diffs chirurgicaux, `ExecutionLoop` exécute avec auto-correction et création de skills. **Pas encore d'agent utilisable** — c'est la couche moteur.

### Seuil 2 — Agent autonome (Phases 4-5)
`PhaseManager` orchestre les 5 phases (revisitables). `WorkflowAgent` + CLI permettent un usage humain direct. Workflow génère un projet complet de "j'ai une idée" jusqu'à "j'ai des tâches en cours d'exécution".

### Seuil 3 — Dogfooding (Phase 6) ⭐
`MCPServer` expose tous les outils. Branchement Claude Code immédiat. **Workflow construit Workflow** à partir de ce moment. C'est le point où la qualité du produit s'auto-vérifie par usage réel.

### Seuil 4 — Proactif (Phase 7)
`DaemonHeartbeat` (briefings + surveillance), `WatchMode` (annotation passive), `ParallelExecutor` (git worktrees pour tâches indépendantes). L'agent devient présent en arrière-plan sans interrompre.

### Seuil 5 — Multi-surface (Phase 8)
VS Code extension, Telegram bot, GitHub integration, REST API — **tous clients du protocole MCP**. Aucune logique métier dupliquée.

### Seuil 6 — Robustesse + Polish (Phases 9-10)
`CodeIndexer` pour gros projets, `WorkflowLibrary` cross-projet, `BreakingChangeDetector`. Puis `workflow doc generate`, `audit`, `estimate`, `onboard`.
