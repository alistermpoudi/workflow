# 📊 PROGRESS.md — Suivi de Progression Workflow

## État Actuel

**Phase courante** : Phase 0 — Vision & Architecture
**Statut** : ✅ Documentation initiale créée
**Prochaine étape** : Démarrer Phase 1 — Foundation

---

## Vue d'Ensemble des Phases

| Phase | Nom | Statut |
|-------|-----|--------|
| 0 | Vision & Architecture | ✅ Complété |
| 1 | Foundation (ProjectMemory, DecisionsLog, SyncChecker) | ⬜ Non démarré |
| 2 | Les 5 Phases Projet | ⬜ Non démarré |
| 3 | Exécution de Base (ExecutionLoop, CLI) | ⬜ Non démarré |
| 4 | MCP Server (Workflow Core) | ⬜ Non démarré |
| 5 | Versioning & Git | ⬜ Non démarré |
| 6 | Robustesse (CodePatcher, CodeIndexer) | ⬜ Non démarré |
| 7 | Interfaces supplémentaires (Telegram, REST) | ⬜ Non démarré |

---

## Phase 1 — Foundation

**Tâches** :

| Tâche | Description | Statut |
|-------|-------------|--------|
| 1.1 | Monorepo init — `package.json`, structure `src/`, config ESLint/Vitest | ⬜ |
| 1.2 | `FileSystem.js` — opérations fichiers sur `.workflow/` | ⬜ |
| 1.3 | `ProjectMemory.js` — CRUD `project.json`, `vision.md`, `features.json`, `tech-stack.json` | ⬜ |
| 1.4 | `TaskManager.js` — CRUD `TASK-XXX.md` + `progress.json` | ⬜ |
| 1.5 | `DecisionsLog.js` — écriture et lecture active du `decisions.log` | ⬜ |
| 1.6 | `SyncChecker.js` — détection state drift + vérification branche Git | ⬜ |

**Pour démarrer** : Lire `.claude/docs/tasks/phase-1-foundation/INDEX.md`

---

## Décisions Techniques Prises

| Date | Décision | Raison |
|------|----------|--------|
| 2026-03-31 | Commencer par Workflow Core (MCP) avant Workflow Agent | Chemin le plus court vers quelque chose d'utilisable — branché sur Claude Code immédiatement |
| 2026-03-31 | CLI MVP avec readline + chalk (pas Ink) | Ink est lourd à déboguer ; garder pour v1.5 après validation MVP |
| 2026-03-31 | Ripgrep subprocess pour CodeIndexer (pas regex custom) | Plus fiable sur de vrais projets |

---

## Dernière Session

**Date** : 2026-03-31
**Travail effectué** : Création du projet — CLAUDE.md, PROGRESS.md, documentation phases 0-7
**Arrêté à** : Fin de la documentation initiale
**Prochaine action** : `cat .claude/docs/tasks/phase-1-foundation/INDEX.md` puis démarrer tâche 1.1
