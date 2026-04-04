# Phase 0 — Architecture Technique

## Vue d'Ensemble

Workflow est une application Node.js structurée en couches clairement séparées. Le principe fondamental : **le core est indépendant de l'interface**. La même logique (`ProjectMemory`, `DecisionsLog`, `ExecutionLoop`) fonctionne en CLI, en MCP, en Telegram ou en API REST.

---

## Architecture en Couches

```
┌─────────────────────────────────────────────────────────────────┐
│  INTERFACES                                                     │
│  CLI (readline)  │  MCP Server  │  Telegram Bot  │  REST API   │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  ORCHESTRATEUR                                                  │
│  WorkflowAgent.js (Agent mode) / PhaseManager.js               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  PHASES                                                         │
│  Discovery → Specification → Validation → Architecture → Exec  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  CORE                                                           │
│  ProjectMemory  │  VersionManager  │  DecisionsLog              │
│  SyncChecker    │  ContextManager  │  TaskManager               │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  OUTILS                                                         │
│  FileSystem  │  GitManager  │  ExecutionLoop                    │
│  CodePatcher │  CodeIndexer │  (v1.5+)                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  LLM                                                            │
│  LLMProvider (abstraction multi-LLM)  │  PromptBuilder          │
└─────────────────────────────────────────────────────────────────┘
                             │
                    Anthropic Claude API
                    (claude-sonnet-4-6 par défaut)
```

---

## Structure des Fichiers Source

```
src/
├── core/
│   ├── WorkflowAgent.js      # Orchestrateur mode Agent (boucle principale)
│   ├── ProjectMemory.js      # Lecture/écriture de tous les fichiers .workflow/
│   ├── VersionManager.js     # Cycle de vie versions, pilote GitManager
│   ├── DecisionsLog.js       # Écriture + lecture active decisions.log + decisions-graph.json
│   ├── ContextManager.js     # Hiérarchie de chargement contexte LLM (scoring de pertinence)
│   ├── PhaseManager.js       # Orchestre les 5 phases (Discovery → Réalisation)
│   ├── SyncChecker.js        # Détection state drift + vérification branche Git + préconditions
│   ├── DaemonHeartbeat.js    # Daemon de surveillance continue + briefings quotidiens (v1.5)
│   └── OnboardingManager.js  # Onboarding nouveau dev depuis .workflow/ (v1.5)
│
├── phases/
│   ├── DiscoveryPhase.js     # Questions → vision.md (capture l'intent utilisateur)
│   ├── SpecificationPhase.js # Propositions → features.json (intent par feature)
│   ├── ValidationPhase.js    # Tâches → versions/ (génère Intent + Préconditions)
│   ├── ArchitecturePhase.js  # Stack → tech-stack.json + TASK-001/002
│   └── ExecutionPhase.js     # Code → validate → corriger (checkPreconditions + FailurePatterns)
│
├── tools/
│   ├── CodePatcher.js        # Diffs chirurgicaux + fallback AST (v1.5)
│   ├── CodeIndexer.js        # Index JSON + variantes LLM + ripgrep (v1.5)
│   ├── ExecutionLoop.js      # build_validate → test → correction (FailurePatterns + loop detection)
│   ├── FailurePatterns.js    # Mémoire des erreurs connues → failure-patterns.json
│   ├── FileSystem.js         # Opérations fichiers (wrapper fs/promises)
│   ├── GitManager.js         # Opérations git (status, checkout, merge, branch)
│   ├── TaskManager.js        # CRUD TASK-XXX.md + progress.json (Intent + Préconditions)
│   ├── WatchMode.js          # Observateur chokidar → .workflow/questions/ (v1.5)
│   └── ConflictResolver.js   # Détecte conflits de décisions entre devs (v1.5)
│
├── interfaces/
│   ├── CLI.js                # readline + chalk (MVP) — watch, daemon, onboard, doc, audit, estimate
│   ├── TelegramBot.js        # node-telegram-bot-api (v2)
│   ├── MCPServer.js          # @modelcontextprotocol/sdk (Workflow Core)
│   ├── RestAPI.js            # Express local (v2)
│   ├── PipeCLI.js            # stdin/stdout pipe (v2)
│   ├── VSCodeExtension/      # Sidebar état projet + annotations inline (v2)
│   └── GitHubIntegration.js  # PR mergée → tâche DONE automatiquement (v2)
│
└── llm/
    ├── LLMProvider.js        # Abstraction — Claude par défaut, multi-LLM futur
    └── PromptBuilder.js      # Construit les prompts avec contexte projet injecté (intent inclus)
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

| Composant | Choix | Version |
|-----------|-------|---------|
| Runtime | Node.js | 22 LTS |
| LLM | `@anthropic-ai/sdk` | latest |
| Modèle défaut | claude-sonnet-4-6 | — |
| MCP | `@modelcontextprotocol/sdk` | latest |
| CLI MVP | `readline` (natif) + `chalk` | — |
| CLI v1.5 | `ink` + `react` | — |
| Telegram | `node-telegram-bot-api` | — |
| Watch mode | `chokidar` | v1.5 |
| GitHub integration | `@octokit/rest` | v2 |
| VS Code extension | VS Code Extension API | v2 |
| AST | `tree-sitter` + grammaires | v1.5 |
| Recherche | `ripgrep` subprocess | — |
| Tests | `vitest` | — |
| Linter | `eslint` + `@antfu/eslint-config` | — |

---

## Roadmap Technique

### MVP (Phase 1-4)
Workflow Core fonctionnel, branché sur Claude Code via MCP. L'utilisateur peut gérer la mémoire projet depuis Claude Code sans construire le Workflow Agent complet.

### V1 (Phase 5)
Versioning Git complet. Workflow Agent autonome utilisable en CLI pour les projets personnels.

### V1.5 (Phase 6)
`CodePatcher` (diffs chirurgicaux + tree-sitter) et `CodeIndexer` (ripgrep + variantes LLM). Robustesse sur les gros projets.

### V2 (Phase 7)
Telegram, REST API, CLI Ink, pipe. Workflow utilisable partout, pas seulement en console locale.
