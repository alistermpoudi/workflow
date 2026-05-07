# 🤖 CLAUDE.md — Guide pour Claude Code

## À propos de ce fichier

Ce fichier contient toutes les instructions et le contexte nécessaires pour travailler efficacement sur **Workflow**. Lis ce fichier en premier avant toute intervention.

---

## 🎯 Qu'est-ce que Workflow ?

**Workflow** est un agent de code pour freelance et développeur indépendant qui **se souvient et accumule de l'expérience**, comme un humain qui a fait 30 projets et applique ses leçons.

### Le Problème à Résoudre

Les agents de code actuels (Cursor, Claude Code, Codex) souffrent de **deux défauts structurels** :

1. **Ils vivent dans le contexte de conversation.** Quand le contexte sature, l'agent perd le fil. L'utilisateur réexplique tout depuis zéro.
2. **Ils oublient tout au projet suivant.** Aucune capitalisation d'expérience. Un dev senior qui a résolu le même problème 10 fois ne le ré-aborde pas comme un junior — l'agent, si.

**Workflow résout les deux.**

### La Solution

Deux mémoires distinctes, persistantes :

1. **Mémoire projet** (`.workflow/` dans le projet) — vision, fonctionnalités, stack, décisions, état d'avancement, contexts actifs.
2. **Mémoire institutionnelle** (`~/.workflow/contexts/` global) — patterns appris à travers tous les projets, organisés en **contexts spécialisés** (mobile.flutter, web.nextjs, backend.fastapi, etc.). Workflow devient plus intelligent à chaque projet.

> **Thèse différenciatrice** : Cursor / Claude Code sont des "assistants experts dans la session". Workflow est un **PM technique avec mémoire institutionnelle**, organisé en cabinet de devs spécialisés.

### Les 7 Piliers Load-Bearing

1. **Protocole `.workflow/`** — schéma versionné lisible par tous les clients (CLI, VS Code, Telegram, web, MCP). Pas un dossier ad-hoc.
2. **Skills + Curator + 4 sources d'apprentissage** — `auto_retry` (échec→succès), `user_explicit` (`workflow teach`), `in_flow_correction` (Ctrl+T), `project_ingestion` (`learn-from`). Niveau `USER_OVERRIDE` au-dessus de HIGH.
3. **Multi-LLM par rôle** — `reasoning` (Opus pour Discovery/Spec/Arch), `code_generation` (DeepSeek pour ExecutionLoop), `fast` (Haiku pour scoring), `curator` (Sonnet pour consolidation), `compression` (Haiku pour résumés).
4. **Décisions actives + graphe rétro-propageant** — `decisions.log` + `decisions-graph.json` avec relations CONTRADICTS/DEPENDS_ON/SUPERSEDES/REFINES. Détecte automatiquement les contradictions et propage aux tâches en cours.
5. **CodePatcher chirurgical** — search/replace blocks dès le départ. Jamais de régénération de fichier entier.
6. **MCP Server comme surface primaire** — toutes les autres surfaces (CLI, VS Code, Telegram, REST) sont des clients du protocole MCP avec `AllowedCommandsPolicy` à 3 niveaux.
7. **Contexts spécialisés** — hiérarchie `_global → mobile → mobile.flutter`, templates bundled, héritage, exportables/installables. Workflow devient un **cabinet de devs spécialisés**.

> Détail complet : `.claude/docs/tasks/phase-0-vision/02-pillars.md`

---

## 🏗️ Deux Modes d'Existence

| Mode | Description | Qui réfléchit |
|------|-------------|---------------|
| **Workflow Agent** | Agent autonome complet — CLI interactive ou Telegram | Workflow (LiteLLM multi-modèles) |
| **Workflow Core** | Gestionnaire de projet pur exposé via MCP | L'agent hôte (Claude Code, Cursor, etc.) |

Dans les deux cas : même cerveau, mêmes fichiers `.workflow/`, mêmes contexts, mêmes outils. Seule la couche d'interaction change.

> ⚠️ **Stratégie de build** : Commencer par **Workflow Core (MCP)** — c'est le chemin le plus court vers quelque chose d'utilisable. Workflow Core branché sur Claude Code donne immédiatement de la valeur, avant même que le Workflow Agent autonome existe. **Seuil dogfooding = fin Phase 6.**

---

## 📁 Structure du Projet

```
workflow/
├── CLAUDE.md                       # CE FICHIER
├── pyproject.toml                  # uv + hatchling + dépendances
├── .python-version                 # "3.12"
├── workflow.config.yaml            # Routage LLM par rôle (Pilier 3)
├── .gitignore
│
├── src/workflow/
│   ├── core/                       # Cerveau cognitif — indépendant de l'interface
│   │   ├── workflow_agent.py       # Orchestrateur mode Agent (Phase 5)
│   │   ├── project_memory.py       # Reference impl du protocole .workflow/
│   │   ├── phase_manager.py        # Orchestre les 6 phases (revisitables + Starter Mode)
│   │   ├── llm_context_loader.py   # Hiérarchie + compression + scoring (renommé depuis ContextManager)
│   │   ├── decisions_log.py        # decisions.log + index SQLite FTS5
│   │   ├── decisions_graph.py      # Graphe + relations + détection contradictions + rétro-propagation
│   │   ├── skill_manager.py        # Skills cross-projet (CRUD + recherche)
│   │   ├── skill_curator.py        # Consolidation périodique via LLM role='curator'
│   │   ├── teach_system.py         # workflow teach / avoid + USER_OVERRIDE
│   │   ├── in_flow_corrector.py    # Ctrl+T → skill USER_OVERRIDE → retry
│   │   ├── context_registry.py     # Contexts hiérarchiques + héritage + templates (Pilier 7)
│   │   ├── onboarding_manager.py   # workflow onboard
│   │   ├── sync_checker.py         # State drift + branche Git + protocole + contradictions
│   │   └── daemon_heartbeat.py     # Daemon proactif (briefings + auto-curator)
│   │
│   ├── phases/                     # Les 6 phases projet (revisitables)
│   │   ├── discovery_phase.py      # Questions → vision.md
│   │   ├── specification_phase.py  # Propositions → features.json
│   │   ├── design_system_phase.py  # Style → design.json + screen-flow.md
│   │   ├── architecture_phase.py   # Stack → tech-stack.json + TASK-001/002
│   │   ├── validation_phase.py    # Tâches → versions/ (mockups + Active Contexts)
│   │   └── execution_phase.py      # Sélection tâche prête → délègue à ExecutionLoop
│   │
│   ├── tools/                      # Outils techniques
│   │   ├── filesystem.py           # WorkflowPaths + opérations async + atomic write
│   │   ├── git_manager.py          # asyncio.create_subprocess_exec
│   │   ├── task_manager.py         # CRUD TASK-XXX.md + progress.json
│   │   ├── version_manager.py     # Cycle de vie versions, pilote GitManager
│   │   ├── code_patcher.py         # Diffs chirurgicaux + tree-sitter (Phase 2 — pas v1.5)
│   │   ├── execution_loop.py       # generate_patch → apply → validate → retry → skill auto
│   │   ├── code_indexer.py         # Index symboles + ripgrep (Phase 9)
│   │   ├── parallel_executor.py    # Git worktrees pour tâches indépendantes (Phase 7)
│   │   ├── project_ingester.py     # workflow learn-from (Phase 9)
│   │   ├── watch_mode.py           # watchfiles → .workflow/questions/ (Phase 7)
│   │   ├── conflict_resolver.py    # Conflits décisions entre devs (Phase 8)
│   │   ├── workflow_library.py     # Patterns cross-projet (Phase 9)
│   │   └── breaking_change_detector.py  # Phase 9
│   │
│   ├── interfaces/                 # Couches d'interaction (clients du protocole MCP)
│   │   ├── cli.py                  # typer + rich + RichIO (Phase 5)
│   │   ├── mcp_server.py           # SDK Python officiel (Phase 6) — surface primaire
│   │   ├── allowed_commands_policy.py  # Whitelist + apprentissage (Phase 6)
│   │   ├── mcp_client.py           # Client MCP réutilisable (Phase 8)
│   │   ├── github_integration.py   # PR mergée → tâche DONE (Phase 8)
│   │   ├── rest_api.py             # FastAPI local (Phase 8)
│   │   ├── telegram_bot.py         # Client MCP wrapper (Phase 8)
│   │   └── vscode_extension/       # TypeScript — VS Code API (Phase 8)
│   │
│   ├── llm/                        # Abstraction LLM
│   │   ├── llm_provider.py         # LiteLLM multi-modèles par rôle
│   │   └── prompt_builder.py       # Prompts avec contexte + skills + decisions injectés
│   │
│   ├── protocol/                   # Pilier 1 — schéma versionné
│   │   ├── version.py              # SCHEMA_VERSION
│   │   ├── validator.py            # Validation JSON Schema
│   │   ├── schemas/                # JSON Schemas (project, features, tech-stack, decisions-graph...)
│   │   └── migrations/             # Migrations entre versions de schéma
│   │
│   └── templates/contexts/         # Pilier 7 — templates bundled
│       ├── _global/
│       ├── mobile/
│       ├── mobile.flutter/
│       ├── mobile.react-native/
│       ├── web/
│       ├── web.nextjs/
│       ├── web.vite-react/
│       ├── desktop/
│       ├── desktop.electron/
│       ├── desktop.tauri/
│       ├── backend/
│       ├── backend.fastapi/
│       └── backend.express/
│
└── .claude/
    ├── PROGRESS.md                 # Suivi de progression — lire en début de session
    └── docs/
        ├── tasks/                  # Tâches détaillées par phase de build (0 à 10)
        │   ├── INDEX.md            # Vue d'ensemble
        │   ├── phase-0-vision/
        │   ├── phase-1-foundation/
        │   ├── phase-2-cognitive/
        │   ├── phase-3-execution-engine/
        │   ├── phase-4-project-phases/
        │   ├── phase-5-agent-cli/
        │   ├── phase-6-mcp/
        │   ├── phase-7-proactive/
        │   ├── phase-8-surfaces/
        │   ├── phase-9-robustness/
        │   └── phase-10-polish/
        └── PROTOCOL.md             # Spec publique du protocole .workflow/
```

---

## 📐 Structure `.workflow/` (générée dans les projets cibles)

```
.workflow/
├── project.json                    # active_contexts: [_global, mobile, mobile.flutter] + schema_version
├── vision.md                       # Sortie Discovery
├── features.json                   # Sortie Specification
├── tech-stack.json                 # Sortie Architecture (build_validate, test, allowed_commands)
├── design.json                     # Sortie DesignSystem (style + couleurs + références)
├── code-index.json                 # Index symboles (Phase 9)
├── decisions.log                   # Journal texte brut — lisible humain
├── decisions-graph.json            # Relations + contradictions + niveaux de confiance
├── allowed-commands.json           # Whitelist + apprentissage (Pilier 6)
├── skills/                         # Skills strictement projet
│   └── *.md
├── questions/                      # WatchMode — questions en attente
│   └── YYYY-MM-DD-Q001.md
├── briefings/                      # DaemonHeartbeat — briefings quotidiens
│   └── YYYY-MM-DD.md
└── versions/
    ├── v1.0/
    │   ├── meta.json               # { title, status, branch, created_at }
    │   ├── progress.json           # { done, pending, failed, deferred }
    │   └── tasks/
    │       ├── TASK-001.md         # Setup + linter (systématique)
    │       ├── TASK-002.md         # Tests + smoke test (systématique)
    │       └── TASK-003.md
    └── v1.5/
```

**Et au niveau global** (`~/.workflow/`) :
```
~/.workflow/
├── config.yaml                     # Routage LLM par défaut
├── ingestions.log                  # Audit trail des project ingestions
└── contexts/                       # Pilier 7 — expertises spécialisées (lazy)
    ├── _global/                    # Universal — créé au boot
    │   ├── config.yaml
    │   ├── skills/
    │   ├── decisions.log
    │   └── teach.md                # USER_OVERRIDE rules
    ├── mobile/                     # Hérite _global
    │   ├── flutter/                # Hérite mobile
    │   └── react-native/
    └── web/
        └── nextjs/
```

---

## 🛠️ Stack Technique

| Composant | Choix | Notes |
|-----------|-------|-------|
| Runtime | Python | 3.12+ |
| Package manager | `uv` | Lockfile + gestion Python version |
| LLM (multi-rôle) | `litellm` | Routage par rôle — Claude, DeepSeek, Ollama |
| Modèle reasoning | `claude-opus-4-7` | Discovery, Specification, Architecture |
| Modèle code | `deepseek-coder-v2` | ExecutionLoop — best HumanEval |
| Modèle fast | `claude-haiku-4-5` | Scoring décisions, préconditions, contradictions |
| Modèle curator | `claude-sonnet-4-6` | Consolidation skills + project ingestion |
| Modèle compression | `claude-haiku-4-5` | Résumés de session |
| MCP | `mcp` (SDK Python officiel Anthropic) | Transport stdio |
| CLI | `typer` + `rich` (RichIO) | Couleurs, panels, prompts |
| Base de données | `aiosqlite` + SQLite FTS5 | DecisionsLog avec recherche plein-texte |
| Fichiers async | `aiofiles` + `pathlib` | Opérations non-bloquantes |
| AST (CodePatcher) | `tree-sitter` Python + grammaires | **Dès Phase 2** — pas v1.5 |
| Recherche code | `ripgrep` (subprocess) | Index + recherche fast |
| Watch mode | `watchfiles` | Surveillance fichiers |
| Git | `asyncio.create_subprocess_exec` | GitManager complet |
| YAML | `pyyaml` | Skills frontmatter, config |
| Tests | `pytest` + `pytest-asyncio` | — |
| Linter | `ruff` | Lint + format |
| Type check | `mypy` strict | — |
| VS Code extension | TypeScript (VS Code API) | Client MCP — Phase 8 |
| Telegram | `python-telegram-bot` | Client MCP — Phase 8 |
| GitHub | `PyGithub` ou `octokit-py` | Client MCP — Phase 8 |
| REST API | FastAPI + uvicorn | Client MCP — Phase 8 |

---

## 📋 Les 11 Phases de Build (0 à 10)

| Phase | Nom | Piliers couverts | Seuil |
|-------|-----|------------------|-------|
| 0 | Vision & Architecture | — (cadre conceptuel) | — |
| 1 | Protocole & Persistence | 1, 2 (foundation), 4, 7 | — |
| 2 | Cerveau Cognitif | 3, 5 | — |
| 3 | Boucle d'Exécution | 2 (sources auto + teach), 5 | — |
| 4 | Phases Projet (revisitables) | Support | — |
| 5 | Agent & CLI | 6 (préparation), 2 (in-flow) | — |
| 6 | **MCP Server** ⭐ | 6 (complet) | **DOGFOODING START** |
| 7 | Couches Proactives | Support 2, 4 | — |
| 8 | Surfaces Tierces | 6 (achevé) | — |
| 9 | Robustesse + Project Ingestion | Renforce 1, étend 2 | — |
| 10 | Polish & Différenciateurs | Met en valeur tous les piliers | — |

> Détails : `.claude/docs/tasks/INDEX.md` + chaque `phase-X-*/INDEX.md`

---

## 🔑 Concepts Clés Non-Négociables

### 1. La Hiérarchie de Chargement de Contexte (`LLMContextLoader`)

À chaque démarrage/reprise, charge dans cet ordre strict :

```
Système (toujours — ~500 tokens max) :
  project.json + vision.md résumé + tech-stack.json + active_contexts

Version active (au switch) :
  meta.json + progress.json (tâches done/pending, sans le contenu)

Tâche courante (au start task) :
  TASK-XXX.md complet
  Fichiers listés dans "Fichiers à modifier" (lecture sélective)
  Skills depuis les contexts actifs (du plus spécifique au plus général)
  Avoidances depuis teach.md des contexts actifs
  Décisions scorées (Score = similarité × récence × scope × confiance ; budget 2000 tokens)

On-demand :
  CodeIndexer.query() pour fonctions pertinentes (Phase 9)
  ContextManager.loadOnDemand(query) pour ripgrep
```

**Ne jamais charger l'intégralité du projet en contexte** — c'est exactement le problème qu'on résout.

### 2. Le `decisions-graph` est Actif, pas Passif

Workflow **consulte** le log avant de coder une tâche impliquant des choix déjà faits (ORM, auth, DB, framework). Et **détecte automatiquement** les contradictions via LLM `role='fast'`. Quand le dev change une décision, **rétro-propagation** vers les tâches `pending` impactées (annotation Journal).

```
Workflow : "J'allais utiliser TypeORM, mais decisions-graph indique Prisma
           choisi le 12 mars (DEC-001, USER_OVERRIDE).
           Je code avec Prisma."
```

### 3. Règle de Granularité Sémantique des Tâches

```
1 tâche Workflow = 1 PR mergeable atomiquement avec ses tests
```

Le nombre de fichiers et la durée sont des indicateurs, pas des règles strictes. Le critère réel est la **cohérence atomique du diff**.

### 4. `AllowedCommandsPolicy` — Sécurité MCP à 3 Niveaux

`MCPServer` exécute via `AllowedCommandsPolicy.authorize(command)` :
- **Niveau 1** : built-in safe (git status, ls, pytest, etc.) → ALLOW direct
- **Niveau 2** : project allowed (`.workflow/allowed-commands.json`) → ALLOW si match
- **Niveau 3** : prompt utilisateur (apprentissage : O / P (préfixe) / E (exact) / N / B (bannir))
- **ALWAYS_DENIED** hardcodé : `rm -rf /`, `sudo rm`, `chmod 777`, `curl | sh`, `dd if=`. Non-overridable.

En CI/non-interactif, les commandes inconnues sont DENY par défaut.

### 5. Versioning Git — Règle "No Stash"

Workflow **ne stashe jamais automatiquement**. Si le répertoire n'est pas propre lors d'un `version switch`, Workflow bloque avec un message explicite et demande à l'utilisateur de commiter.

### 6. Le Format des Tâches Auto-Suffisantes

Chaque `TASK-XXX.md` doit pouvoir être exécuté **sans historique de conversation** :

```markdown
# TASK-XXX : Titre
## Version : vX.Y

## Contexte Projet
Application, stack, phase courante

## Active Contexts
[_global, mobile, mobile.flutter]

## User Story
EN TANT QUE / JE VEUX / AFIN DE

## Intent
Pourquoi l'utilisateur veut vraiment cette fonctionnalité — guide les décisions ambiguës.

## Préconditions
- filesExist: [...]
- tasksCompleted: [...]
- branch: "workflow/vX.Y"

## Dépendances
- TASK-001 ✅
- TASK-002 ✅

## Fichiers à créer / modifier
- src/foo.py [CRÉER]
- src/bar.py [MODIFIER]

## Critères d'acceptation
- [ ] Critère 1
- [ ] Critère 2

## Mockup UI
(ASCII art conforme à design.json — ou "(aucune interface)" pour backend)

## Journal
[date] Reportée depuis vX.X — raison : ...
[date] Tentative partielle : fichier src/foo.py créé, manque ...

## Statut
⬜ EN ATTENTE | 🔄 EN COURS | ✅ TERMINÉ | ❌ REPORTÉ
```

> **Le champ `Journal`** est rempli automatiquement par Workflow.
> **Le champ `Mockup UI`** est généré par `ValidationPhase` en respectant le style de `design.json`.
> **Le champ `Active Contexts`** est lu depuis `project.json#active_contexts`.

### 7. TASK-001 et TASK-002 sont Systématiques

Les deux premières tâches de toute v1.0 sont imposées par `ArchitecturePhase` :
- **TASK-001** : Setup du projet et configuration du linter
- **TASK-002** : Configuration du framework de tests avec un premier test de smoke

### 8. Reprise après Context Overflow

```
SyncChecker au démarrage :
0. Valide la conformité au protocole .workflow/ (JSON Schema)
1. Vérifie que la branche Git = version active dans .workflow/
2. Compare état du repo avec progress.json
3. Détecte fichiers modifiés manuellement → soumet diff au LLM
4. Alerte si contradictions actives dans decisions-graph
5. Annonce l'état exact et demande confirmation avant de reprendre
```

### 9. Le Daemon est Toujours Présent

`DaemonHeartbeat` tourne en arrière-plan (launchd/systemd). Il :
- Envoie un briefing au démarrage de la journée dans `.workflow/briefings/`
- Surveille les builds CI
- Alerte sur les contradictions du decisions-graph
- **Lance automatiquement le `SkillCurator`** quand ≥10 nouveaux skills depuis la dernière run
- Propose la prochaine tâche quand une est terminée

Il ne bloque jamais l'utilisateur.

### 10. Watch Mode — Annotation Sans Interruption

`WatchMode` (via `watchfiles`) observe les fichiers du projet. Quand un fichier est créé/modifié hors du scope d'une tâche, il crée un fichier question dans `.workflow/questions/`. L'utilisateur répond en éditant le fichier. Zéro interruption.

### 11. `workflow onboard` — Onboarding Instantané

Un nouveau développeur lance `workflow onboard`. Workflow lit tout le `.workflow/` (+ contexts actifs + skills accumulés) et génère : résumé du projet, stack expliquée, état d'avancement, 5 décisions clés HIGH-confidence, conventions équipe, prochaine tâche prête, top skills réutilisables, contradictions à connaître. **Objectif : autonomie en 30 secondes.**

### 12. Les 4 Sources d'Apprentissage (Pilier 2 complet)

Workflow apprend par 4 voies cumulatives, du passif au correctif :

1. **`auto_retry`** — `ExecutionLoop` corrige une erreur après retry → skill auto (`MEDIUM`)
2. **`user_explicit`** — `workflow teach` / `workflow avoid` → skill `USER_OVERRIDE` (le plus haut)
3. **`in_flow_correction`** — Ctrl+T pendant que l'agent code → skill `USER_OVERRIDE` + retry
4. **`project_ingestion`** — `workflow learn-from <path>` → skills `HIGH` après review

`SkillCurator` consolide périodiquement : déduplique, promeut au global les patterns confirmés, archive les skills inutilisés. Les `USER_OVERRIDE` ne sont jamais archivés sans confirmation.

### 13. Les Contexts — Pilier 7

Workflow n'a pas une mémoire indifférenciée — il a des **expertises spécialisées** :

```
~/.workflow/contexts/
├── _global/                ← seul créé au boot
├── mobile/                 ← lazy : créé au workflow init Flutter
│   └── flutter/            ← idem (héritage de mobile)
├── web/
│   └── nextjs/
└── backend/
    └── fastapi/
```

- **Templates bundled** dans le package — instantiated lazy au `workflow init`
- **Auto-détection** depuis `pubspec.yaml` / `package.json` / `Cargo.toml` etc.
- **Multi-contexts actifs** par projet (`active_contexts: [_global, web, web.nextjs]`)
- **Héritage** : skills/decisions résolus du plus spécifique au plus général
- **Cross-context promotion** : Curator détecte un pattern présent dans 2+ contexts → propose vers le parent commun
- **Portables** : `workflow context export mobile.flutter > my.tar.gz`

### 14. Design Style — Capturé Dès Discovery, Phase Dédiée

`DiscoveryPhase` demande obligatoirement le style de design (minimaliste, material, glassmorphism, néomorphisme, brutaliste, etc.). `DesignSystemPhase` (Phase 4 du build) génère `design.json` + `design-system.json` + `screen-flow.md`. `ValidationPhase` les utilise pour produire des **mockups ASCII conformes** dans chaque tâche UI.

```json
// design.json — exemple
{
  "style": "minimaliste",
  "styleLabel": "Minimaliste",
  "colorScheme": "clair",
  "references": ["Linear", "Stripe"],
  "customNotes": null,
  "collectedAt": "2026-04-05T10:23:00Z"
}
```

---

## 🔗 Dépendances Entre Phases de Build

```
Phase 0 (Vision & Architecture — docs)
         │
         ▼
Phase 1 (Protocole & Persistence — protocol, skills, decisions-graph, contexts, sync)
         │
         ▼
Phase 2 (Cerveau Cognitif — LLMProvider, PromptBuilder, LLMContextLoader, CodePatcher)
         │
         ▼
Phase 3 (Boucle d'Exécution — ExecutionLoop, SkillCurator, TeachSystem, extensions)
         │
         ▼
Phase 4 (Phases Projet revisitables — PhaseManager, Discovery → Active)
         │
         ▼
Phase 5 (Agent & CLI — WorkflowAgent, CLI, InFlowCorrector)
         │
         ▼
Phase 6 (MCP Server) ◀── 🌟 SEUIL DOGFOODING — Workflow construit Workflow
         │
         ▼
Phase 7 (Couches Proactives — Daemon, Watch, ParallelExecutor)
         │
         ▼
Phase 8 (Surfaces Tierces — VS Code, GitHub, Telegram, REST)
         │
    ┌────┴────┐
    ▼         ▼
Phase 9    Phase 10
(Robustesse + Ingestion)  (Polish — doc, audit, estimate, onboard)
```

---

## ⚠️ Règles NON-NÉGOCIABLES

1. **`AllowedCommandsPolicy` à 3 niveaux** — jamais de commande shell hors built-in safe sans approbation explicite (et apprentissage)
2. **Jamais de stash automatique** — bloquer et expliquer si répertoire non propre
3. **Toujours consulter `decisions-graph`** avant de coder une tâche — détecter contradictions actives
4. **Une seule version ACTIVE à la fois** — refuser de démarrer une v1.5 si v1.0 n'est pas COMPLETED
5. **`LLMContextLoader` charge en hiérarchie stricte** — jamais tout le projet en contexte
6. **Tests obligatoires** — `ExecutionLoop` ne marque pas une tâche `done` si les tests échouent
7. **Les fichiers `.workflow/` sont la source de vérité** — Git est synchronisé depuis Workflow, pas l'inverse
8. **`USER_OVERRIDE` ne sont jamais archivés sans confirmation** — skills/decisions édictés par le dev sont sacrés
9. **Jamais de régénération de fichier complet** — toujours via `CodePatcher` (search/replace blocks ou AST)
10. **Le protocole `.workflow/` est versionné** — `schema_version` requis, migrations automatiques, validation au boot

---

## 📊 Outils MCP Exposés (Workflow Core, Phase 6)

```
── Phase Projet ──────────────────────────────────────────────
workflow_start_project(description)
workflow_save_discovery(answers)
workflow_propose_features()
workflow_save_features(validated)
workflow_generate_tasks()
workflow_validate_task(task_id, approved)
workflow_set_tech_stack(stack)

── Gestion des Versions ──────────────────────────────────────
workflow_version_list()
workflow_version_create(name, description)
workflow_version_switch(version)              # bloque si repo non propre
workflow_version_add_task(version, task)
workflow_version_hotfix(name, reason)         # bloque si repo non propre
workflow_version_complete()

── Contexte & Exécution ──────────────────────────────────────
workflow_get_current_task()
workflow_get_project_context()
workflow_search_codebase(query)
workflow_mark_task_done(task_id)
workflow_mark_task_failed(task_id, reason)
workflow_log_decision(task_id, decision, reason, scope, confidence)
workflow_get_decision_graph()
workflow_run_command(command)                 # via AllowedCommandsPolicy
workflow_approve_command(command, scope)
workflow_correct(task_id, correction)         # in-flow correction (Phase 5)

── Apprentissage (Pilier 2 — 4 sources) ──────────────────────
workflow_teach(rule, context, tags, visibility)
workflow_avoid(rule, reason, context, visibility)
workflow_learn_from(project_path, target_context, learn, ignore, dry_run)
workflow_curate_skills(dry_run)
workflow_list_skills(context)
workflow_remove_teach_rule(rule_name)

── Contexts (Pilier 7) ───────────────────────────────────────
workflow_context_list()
workflow_context_list_available()
workflow_context_create_from_template(name)
workflow_context_create_custom(name, parent, description)
workflow_context_activate(name, scope='project')
workflow_context_export(name, path)
workflow_context_install(archive_path)
workflow_context_fork(source, target)
workflow_context_delete(name)
workflow_context_promote_skill(skill_id, from_context, to_context)

── Polish & Différenciateurs ─────────────────────────────────
workflow_doc_generate()
workflow_audit()
workflow_estimate(version)
workflow_onboard()
workflow_daily_briefing()
```

---

## 🚀 Roadmap par Seuils de Capacité

### Seuil 1 — Foundations (Phases 1-3)
Cerveau cognitif fonctionnel. `LLMProvider` route par rôle, `LLMContextLoader` charge en hiérarchie + compresse + résout contexts actifs, `CodePatcher` applique des diffs chirurgicaux, `ExecutionLoop` exécute avec auto-correction et création de skills (sources auto_retry + user_explicit). **Pas encore d'agent utilisable** — c'est la couche moteur.

### Seuil 2 — Agent autonome (Phases 4-5)
`PhaseManager` orchestre les 6 phases (revisitables + Starter Mode). `WorkflowAgent` + CLI permettent un usage humain direct. `InFlowCorrector` capture les Ctrl+T. Workflow génère un projet complet de "j'ai une idée" jusqu'à "j'ai des tâches en cours d'exécution".

### Seuil 3 — Dogfooding (Phase 6) ⭐
`MCPServer` expose tous les outils. `AllowedCommandsPolicy` à 3 niveaux. Branchement Claude Code immédiat. **Workflow construit Workflow** à partir de ce moment. C'est le point où la qualité du produit s'auto-vérifie par usage réel.

### Seuil 4 — Proactif (Phase 7)
`DaemonHeartbeat` (briefings + auto-curator), `WatchMode` (annotation passive), `ParallelExecutor` (git worktrees). L'agent devient présent en arrière-plan sans interrompre.

### Seuil 5 — Multi-surface (Phase 8)
VS Code extension, Telegram bot, GitHub integration, REST API — **tous clients du protocole MCP**. `MCPClient` réutilisable. Aucune logique métier dupliquée.

### Seuil 6 — Robustesse + Ingestion (Phase 9)
`CodeIndexer` pour gros projets, `WorkflowLibrary` cross-projet, `BreakingChangeDetector`, **`ProjectIngester` (`workflow learn-from`)** — la 4ème source d'apprentissage qui valorise les projets passés.

### Seuil 7 — Polish & Différenciateurs (Phase 10)
`workflow doc generate`, `workflow audit`, `workflow estimate`, `workflow onboard`. Les commandes qui rendent Workflow unique et visible.

---

## 🔄 Suivi de Progression

### Au début de chaque session :
1. **Lire `.claude/PROGRESS.md`** — indique exactement où on s'est arrêté
2. Identifier la phase et la tâche en cours
3. Lire le fichier de tâche correspondant dans `.claude/docs/tasks/phase-X-*/`
4. Reprendre exactement là où on s'est arrêté

### Après avoir complété une tâche :
1. Mettre à jour `.claude/PROGRESS.md`
2. Committer le code + `PROGRESS.md` ensemble

📄 **Fichier de suivi** : `.claude/PROGRESS.md`
📄 **Vue d'ensemble** : `.claude/docs/tasks/INDEX.md`

---

## ⚠️ INSTRUCTION GLOBALE

Les extraits de code dans les fichiers de tâches sont **indicatifs**, pas complets. Leur rôle est d'illustrer la structure et la méthodologie. C'est à Claude Code d'écrire le code complet, fonctionnel et robuste en s'en inspirant.

---

*Workflow est développé par Alister. Le but : construire l'outil qu'on aurait voulu avoir pour construire LinkStream — un agent qui ne perd jamais le fil, version par version, du début jusqu'à la livraison, et qui devient plus intelligent à chaque projet grâce à sa mémoire institutionnelle organisée en cabinet d'expertises spécialisées.*
