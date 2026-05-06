# 📊 PROGRESS.md — Suivi de Progression Workflow

## État Actuel

**Phase courante** : Phase 1 — Foundation (prête à implémenter)
**Statut** : ✅ Toutes les spécifications `.claude/docs/tasks/` mises à jour pour Python
**Prochaine étape** : Implémenter Phase 1 — Foundation

---

## Décision Stratégique — Refonte Python (2026-05-02)

Le projet a été refondu pour utiliser **Python 3.12+** à la place de Node.js, suite à l'analyse du projet Hermes (NousResearch). Raisons principales :

1. **LiteLLM** — interface unifiée pour 100+ LLMs (Claude, DeepSeek, Llama, Ollama...) sans adapters custom
2. **Modèles par rôle** — `reasoning` (Opus), `code_generation` (DeepSeek Coder), `fast` (Haiku), `compression` (Haiku)
3. **SkillManager** (Hermes-inspired) — accumulation d'expérience cross-projet via SKILL.md
4. **SQLite FTS5** — DecisionsLog robuste avec recherche plein-texte (remplace text file fragile)
5. **Python mcp SDK** — MCP officiel Anthropic pour Python
6. **`typer` + `rich`** — CLI propre avec `RichIO` injectable dans toutes les phases
7. **Compression de contexte 3 phases** (Hermes-inspired) — précompression → protection head/tail → résumé LLM

---

## Vue d'Ensemble des Phases

| Phase | Nom | Statut |
|-------|-----|--------|
| 0 | Vision & Architecture | ✅ Complété |
| 1 | Foundation (ProjectMemory, DecisionsLog SQLite, SyncChecker, SkillManager) | 📋 Spécifié — prêt à coder |
| 2 | Les 5 Phases Projet | 📋 Spécifié — prêt à coder |
| 3 | Exécution de Base (ExecutionLoop, CLI typer+rich, Daemon, Watch) | 📋 Spécifié — prêt à coder |
| 4 | MCP Server (Workflow Core) | 📋 Spécifié — prêt à coder |
| 5 | Présence & Intégrations (VS Code, GitHub, Onboarding) | 📋 Spécifié |
| 6 | Robustesse (CodePatcher, CodeIndexer, WorkflowLibrary) | 📋 Spécifié |
| 7 | Génération & Audit (doc, audit, estimate) | 📋 Spécifié |
| 8 | Écosystème (Telegram, REST, workflow-hub) | ⬜ Non spécifié |

---

## Phase 1 — Foundation

**Pour démarrer** : Lire `.claude/docs/tasks/phase-1-foundation/INDEX.md`

| Tâche | Description | Statut |
|-------|-------------|--------|
| 1.0 | `skill_manager.py` — système de skills cross-projet (SKILL.md) | ⬜ |
| 1.1 | Monorepo init — `pyproject.toml`, structure `src/workflow/`, ruff, pytest | ⬜ |
| 1.2 | `filesystem.py` — `WorkflowPaths` + opérations fichiers async | ⬜ |
| 1.3 | `project_memory.py` — CRUD `project.json`, `vision.md`, `features.json`, `tech-stack.json`, `design.json` | ⬜ |
| 1.4 | `task_manager.py` — CRUD `TASK-XXX.md` + `progress.json` | ⬜ |
| 1.5 | `decisions_log.py` — SQLite FTS5, écriture et lecture active | ⬜ |
| 1.6 | `sync_checker.py` + `git_manager.py` — state drift + vérification branche | ⬜ |

---

## Phase 2 — Les 7 Phases Projet

| Tâche | Description | Statut |
|-------|-------------|--------|
| 2.1 | `llm_provider.py` + `prompt_builder.py` — LiteLLM multi-modèles par rôle | ⬜ |
| 2.2 | `context_manager.py` — hiérarchie + compression 3 phases | ⬜ |
| 2.3 | `phase_manager.py` — DISCOVERY→**DESIGN**→SPECIFICATION→ARCHITECTURE→VALIDATION→ACTIVE | ⬜ |
| 2.4 | `discovery_phase.py` + `specification_phase.py` | ⬜ |
| 2.5 | `validation_phase.py` + `architecture_phase.py` | ⬜ |
| 2.6 | `execution_phase.py` — sélection tâche + délégation à ExecutionLoop | ⬜ |
| 2.7 | `design_system_phase.py` + `design_reviewer.py` — design system complet + review UI | ⬜ |

---

## Phase 3 — Exécution de Base

| Tâche | Description | Statut |
|-------|-------------|--------|
| 3.1 | `execution_loop.py` — génère code, applique, valide, corrige (max 3 retries) + skill auto-create | ⬜ |
| 3.2 | `cli.py` — typer + rich + `RichIO` | ⬜ |
| 3.3 | `workflow_agent.py` — orchestrateur principal (Agent mode) | ⬜ |
| 3.4 | `daemon_heartbeat.py` — subprocess.Popen start_new_session + briefing async | ⬜ |
| 3.5 | `watch_mode.py` — `watchfiles.awatch` + fichiers questions | ⬜ |
| 3.6 | `parallel_executor.py` — git worktrees + asyncio.gather + merge auto | ⬜ |
| 3.7 | ExecutionLoop extensions — TDD + Security Review + Architecture Review + Dependency Intelligence | ⬜ |

---

## Phase 4 — MCP Server

| Tâche | Description | Statut |
|-------|-------------|--------|
| 4.1 | `mcp_server.py` — Python mcp SDK, transport stdio, tous les outils | ⬜ |
| 4.2 | `version_manager.py` + `git_manager.py` complet | ⬜ |

---

## Phase 5 — Présence & Intégrations (ajouts)

| Tâche | Description | Statut |
|-------|-------------|--------|
| 5.5 | `zero_pr.py` — commit conventionnel + push + PR GitHub/GitLab auto | ⬜ |

---

## Phase 6 — Robustesse (ajouts)

| Tâche | Description | Statut |
|-------|-------------|--------|
| 6.5 | `breaking_change_detector.py` — analyse impact + tests de régression pré-codage | ⬜ |

---

## Décisions Techniques (session 2026-05-02)

| Date | Décision | Raison |
|------|----------|--------|
| 2026-05-02 | Python 3.12+ remplace Node.js | Écosystème IA plus riche (LiteLLM, embeddings, tree-sitter) |
| 2026-05-02 | LiteLLM pour l'abstraction LLM | 100+ providers, fallback natif, modèle par rôle |
| 2026-05-02 | Modèles par rôle (reasoning/code_generation/fast/compression) | Optimisation coût/qualité selon la tâche cognitive |
| 2026-05-02 | SQLite FTS5 pour DecisionsLog | Recherche full-text robuste, WAL mode, remplacement du fichier texte fragile |
| 2026-05-02 | SkillManager Hermes-inspired | Auto-accumulation d'expérience après retry réussi, zéro coût (SKILL.md local) |
| 2026-05-02 | Compression contexte 3 phases (Hermes) | Évite la saturation sur sessions longues sans perdre les décisions critiques |
| 2026-05-02 | `typer` + `rich` remplace readline + chalk | CLI idiomatique Python, `RichIO` injectable dans toutes les phases |
| 2026-05-02 | `watchfiles` remplace chokidar | Python natif, async avec `awatch` |

---

## Dernière Session

**Date** : 2026-05-03
**Travail effectué** : Ajout de 8 features game-changing dans les spécifications
**Nouveaux fichiers de spec créés** :
- `phase-2-phases/07-design-system.md` — `DesignSystemPhase` + `DesignReviewer` + `design-system.json` + `screen-flow.md` ✅
- `phase-3-execution/06-parallel-executor.md` — `ParallelExecutor` git worktrees + asyncio.gather ✅
- `phase-3-execution/07-execution-loop-extensions.md` — TDD + Security Review + Architecture Review + Dependency Intelligence ✅
- `phase-5-mcp-interfaces/05-zero-to-pr.md` — `ZeroPR` commits conventionnels + PR GitHub/GitLab ✅
- `phase-6-robustness/05-breaking-change-detector.md` — `BreakingChangeDetector` + tests de régression ✅
**INDEX mis à jour** : phases 2, 3, 5, 6 ✅
**CLAUDE.md** : stack + phases mis à jour ✅
**Arrêté à** : Toutes les spécifications complètes. Prêt pour l'implémentation.
**Prochaine action** : Démarrer Phase 1 — lire `.claude/docs/tasks/phase-1-foundation/INDEX.md`
