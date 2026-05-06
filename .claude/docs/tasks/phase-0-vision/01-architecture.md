# Phase 0 — Architecture Technique

## Vue d'Ensemble

Workflow est une application **Python 3.12+** structurée en couches clairement séparées. Le principe fondamental : **le core est indépendant de l'interface**. La même logique (`ProjectMemory`, `DecisionsLog`, `ExecutionLoop`) fonctionne en CLI, en MCP, en Telegram ou en API REST. Les interfaces non-Python (VS Code = TypeScript obligatoire) sont des **clients du protocole MCP**, pas des duplications du core.

> Voir aussi : [`02-pillars.md`](02-pillars.md) — détail des 6 bets architecturaux load-bearing.

## Les 6 Piliers Architecturaux

Tout le découpage de phases ci-dessous découle de ces 6 piliers :

| # | Pilier | Phase de build |
|---|--------|----------------|
| 1 | `.workflow/` comme **protocole versionné** (pas dossier ad-hoc) | Phase 1 |
| 2 | Boucle **Skills + Curator** (mémoire institutionnelle cross-projet) | Phases 1 + 3 |
| 3 | **Multi-LLM par rôle** (`reasoning`, `code_generation`, `fast`, `curator`, `compression`) | Phase 2 |
| 4 | **Décisions actives + graphe rétro-propageant** (CONTRADICTS, SUPERSEDES…) | Phase 1 |
| 5 | **CodePatcher chirurgical** dès le départ (jamais de fichier complet régénéré) | Phase 2 |
| 6 | **MCP Server** comme surface primaire (CLI/VS Code/Telegram = clients du protocole) | Phase 6 |

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
│  ProjectMemory  │  ContextManager (tier + compression Hermes-style)     │
│  PhaseManager   │  DecisionsLog + DecisionsGraph (rétro-propagation)    │
│  SkillManager   │  SkillCurator   │  SyncChecker  │  TaskManager        │
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
│   # { name, description, createdAt, currentVersion, status }
│
├── vision.md
│   # Sortie Phase 1 — description libre de l'application
│
├── features.json
│   # Sortie Phase 2 — liste des fonctionnalités validées par version
│   # { "v1.0": [{ id, name, description, priority }], "v1.5": [...] }
│
├── tech-stack.json
│   # Sortie Phase 4 — stack + commandes + sécurité
│   # { language, framework, database, build_validate, test, allowed_commands[] }
│
├── code-index.json
│   # Mis à jour en continu pendant la réalisation
│   # { "src/auth.js": [{ name: "login", type: "function", line: 42 }] }
│
├── decisions.log
│   # Journal actif texte brut — une entrée par décision technique
│   # [date] [TASK-XXX] Décision : ... / Raison : ...
│
├── decisions-graph.json
│   # Relations entre décisions : CONTRADICTS, DEPENDS_ON, SUPERSEDES, REFINES
│   # Niveaux de confiance : HIGH | MEDIUM | LOW
│   # Détection de contradictions actives
│
├── failure-patterns.json
│   # Erreurs connues + solutions cross-tâches (appris par FailurePatterns.js)
│   # [{ fingerprint, fix, occurrences, lastSeen, learnedAt }]
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
    │   ├── meta.json         # { title, description, status, branch, createdAt }
    │   ├── progress.json     # { done: [], pending: [], failed: [] }
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

---

## `ContextManager` — Hiérarchie de Chargement

C'est le composant le plus critique pour ne pas reproduire le problème du context overflow.

```javascript
// Ordre de chargement strict — ne jamais déroger

class ContextManager {
  // Niveau 1 : Système — toujours chargé (~500 tokens max)
  async loadSystemContext() {
    return {
      project: await ProjectMemory.getProjectSummary(),  // nom, stack, version active
      techStack: await ProjectMemory.getTechStack(),
    };
  }

  // Niveau 2 : Version active — chargé au switch de version
  async loadVersionContext(version) {
    return {
      meta: await ProjectMemory.getVersionMeta(version),
      progress: await ProjectMemory.getProgress(version), // tâches done/pending — PAS le contenu
    };
  }

  // Niveau 3 : Tâche courante — chargé au start task
  async loadTaskContext(taskId) {
    const task = await TaskManager.getTask(taskId);
    const relevantFiles = await FileSystem.readSelective(task.filesToModify);
    // Scoring de pertinence : remplace le keyword matching statique
    // Score = similarité(décision, tâche.title + tâche.intent) × récence × confiance
    // Budget tokens = 2000 | Seuil = 0.4
    const scoredDecisions = await this._loadScoredDecisions(task);
    return { task, relevantFiles, relevantDecisions: scoredDecisions };
  }

  // Niveau 4 : On-demand — uniquement si nécessaire
  async loadOnDemand(query) {
    return CodeIndexer.search(query); // (v1.5)
  }
}
```

**Règle** : Charger uniquement le niveau nécessaire pour l'action en cours. Ne jamais passer de niveau 4 en "contexte de base".

**Scoring de pertinence** : Le `ContextManager` ne charge pas les décisions mécaniquement. Il calcule `Score = similarité × récence × confiance` et n'inclut que les décisions avec score ≥ 0.4, dans la limite d'un budget de 2000 tokens. Le champ `intent` de la tâche est inclus dans le texte de référence pour le scoring.

---

## `SyncChecker` — Détection du State Drift

Au démarrage de chaque session :

```javascript
class SyncChecker {
  async check() {
    // 1. Vérifier cohérence branche Git / version active
    const activeBranch = await GitManager.currentBranch();
    const activeVersion = await ProjectMemory.getActiveVersion();

    if (activeBranch !== `workflow/${activeVersion}`) {
      return {
        type: 'BRANCH_MISMATCH',
        message: `Branche Git "${activeBranch}" ≠ version active "${activeVersion}"`
      };
    }

    // 2. Détecter fichiers modifiés manuellement depuis la dernière session
    const lastSession = await ProjectMemory.getLastSessionTimestamp();
    const modifiedFiles = await GitManager.getModifiedSince(lastSession);

    if (modifiedFiles.length > 0) {
      // 3. Soumettre le diff au LLM pour analyse sémantique
      const diff = await GitManager.getDiff(modifiedFiles);
      return { type: 'MANUAL_CHANGES', files: modifiedFiles, diff };
    }

    return { type: 'CLEAN' };
  }
}
```

---

## `ExecutionLoop` — Boucle d'Auto-Correction

```javascript
class ExecutionLoop {
  async executeTask(task) {
    // Consulter decisions.log scoré avant de commencer
    const decisions = await ContextManager.loadTaskContext(task.id); // scoring activé
    const errorHistory = [];

    let attempts = 0;
    while (attempts < 3) {
      attempts++;

      // Chercher un pattern d'échec connu AVANT de générer le code
      const knownFix = await FailurePatterns.match(lastError?.output);
      if (knownFix) context.knownFix = knownFix; // injecté dans le prompt

      // Générer le code (avec intent injecté dans le prompt)
      const code = await LLMProvider.generateCode(task, context, decisions);
      await FileSystem.applyCode(code);

      // Valider
      const buildResult = await this.runBuildValidate();
      if (!buildResult.success) {
        errorHistory.push(buildResult.output);
        // Détecter boucle stérile (80% de chevauchement sur les 2 dernières erreurs)
        if (this._isLooping(errorHistory)) {
          return { success: false, escalate: true, reason: 'LOOP_DETECTED' };
        }
        continue;
      }

      const testResult = await this.runTests();
      if (!testResult.success) {
        errorHistory.push(testResult.output);
        if (this._isLooping(errorHistory)) {
          return { success: false, escalate: true, reason: 'LOOP_DETECTED' };
        }
        continue;
      }

      // Succès — apprendre des patterns d'échec si retry
      if (errorHistory.length > 0) {
        await FailurePatterns.learnFromSuccess(errorHistory, code.response);
      }
      await TaskManager.markDone(task.id);
      await DecisionsLog.logDecisions(task.id, code.decisions);
      return { success: true };
    }

    // Échec après 3 tentatives → escalade
    return {
      success: false,
      escalate: true,
      context: await this.buildEscalationContext(task, attempts)
    };
  }
}
```

---

## `MCPServer` — Outils Exposés avec Sécurité

```javascript
// Validation stricte — s'applique à TOUTES les interfaces
function validateCommand(cmd, techStack) {
  if (!techStack.allowed_commands.includes(cmd)) {
    logger.warn(`Commande rejetée : ${cmd}`);
    throw new Error(
      `"${cmd}" non autorisée. Seules les commandes de tech-stack.json#allowed_commands sont acceptées.`
    );
  }
}

// Outils MCP exposés (liste complète dans CLAUDE.md)
const tools = [
  'workflow_start_project',
  'workflow_save_discovery',
  'workflow_propose_features',
  'workflow_save_features',
  'workflow_generate_tasks',
  'workflow_validate_task',
  'workflow_set_tech_stack',
  'workflow_version_list',
  'workflow_version_create',
  'workflow_version_switch',    // bloque si repo non propre
  'workflow_version_add_task',
  'workflow_version_hotfix',    // bloque si repo non propre
  'workflow_version_complete',
  'workflow_get_current_task',
  'workflow_get_project_context',
  'workflow_search_codebase',
  'workflow_mark_task_done',
  'workflow_mark_task_failed',
  'workflow_log_decision',
];
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
Le cerveau cognitif fonctionne. `LLMProvider` route par rôle, `ContextManager` charge en hiérarchie + compresse, `CodePatcher` applique des diffs chirurgicaux, `ExecutionLoop` exécute avec auto-correction et création de skills. **Pas encore d'agent utilisable** — c'est la couche moteur.

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
