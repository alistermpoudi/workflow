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
| 3 | Exécution de Base (ExecutionLoop, CLI, Daemon, Watch) | ⬜ Non démarré |
| 4 | MCP Server (Workflow Core) | ⬜ Non démarré |
| 5 | Présence & Intégrations (VS Code, GitHub, Onboarding) | ⬜ Non démarré |
| 6 | Robustesse (CodePatcher, CodeIndexer, WorkflowLibrary) | ⬜ Non démarré |
| 7 | Génération & Audit (doc, audit, estimate) | ⬜ Non démarré |
| 8 | Écosystème (Telegram, REST, workflow-hub) | ⬜ Non démarré |

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
| 2026-04-03 | `decisions.log` (texte) + `decisions-graph.json` (relations) en parallèle | Log lisible humain + machine ; détection de contradictions possible sans parser le texte |
| 2026-04-03 | Scoring de pertinence dans `ContextManager` (remplace keyword matching statique) | Évite de saturer le contexte avec des décisions non pertinentes sur les gros projets |
| 2026-04-03 | Détection de boucle dans `ExecutionLoop` (80% de similarité entre erreurs) | Évite de brûler 3 tentatives LLM sur le même bug ; escalade plus tôt |
| 2026-04-03 | `WorkflowLibrary` en Phase 6 (cross-project learning) | Capitaliser les patterns validés sans complexifier le MVP |
| 2026-04-03 | Mode `workflow watch` (annotation passive par fichiers) | Capturer les décisions manuelles sans interrompre l'utilisateur |

---

## Dernière Session

**Date** : 2026-04-04
**Travail effectué** : Audits v3 + v4 — correction de tous les bugs identifiés dans la documentation
**Bugs corrigés** :
- **P0** Bug A : `WatchMode._applyAnswer()` implémentée (nouvelle méthode statique complète)
- **P0** P0-1 : `PHASE_ORDER` corrigé → ARCHITECTURE avant VALIDATION (tech-stack toujours disponible)
- **P1** Bug B : `complete(opts)` propagé depuis MCPServer → WorkflowAgent → VersionManager (force: true fonctionnel)
- **P1** Bug C : `WorkflowLibrary._renderPattern(task)` implémentée
- **P1** Bug D : `GitManager.branchExists()` migré vers `runSafe()` (anti-injection)
- **P1** Bug E : `WatchMode.processAnswers()` prend uniquement `projectRoot`, instancie ses deps en interne
- **P2** Bug F : `DiscoveryPhase` passe les corrections au LLM (ne les perd plus)
- **P2** Bug G : `injectFoundationTasks()` vérifie pending + done + failed (pas seulement pending)
- **P2** Bug H : `JSON.parse` après corrections wrappé en try/catch dans Spec et Validation
- **P3** P3-1 : Commentaires PhaseManager corrigés (Phase 3, pas Phase 6)
- **P3** P3-2 : `DaemonHeartbeat._checkTaskCompletion()` implémentée avec comparaison d'état
- **P3** P3-3 : `WorkflowLibrary._exists()` utilise `access()` au lieu de `readFile()`
- **P1** Bug BB : `VersionManager.switch()` appelle `saveVersionMeta(..., ACTIVE)` — règle "1 ACTIVE à la fois" réellement enforced
- **P2** Bug CC : `SyncChecker.check()` lit `meta.branch` — plus de faux BRANCH_MISMATCH sur les hotfixes
- **P2** Bug DD : `WatchMode._detectNewImports()` lit `package.json#dependencies` (plus `tech-stack.json#dependencies` inexistant)
- **P2** Bug EE : `WatchMode._onFileChanged()` implémentée — détecte les modifications sur fichiers de tâches terminées
- **P3** Bug GG : Commentaires harmonisés (Phase 3 pour Daemon/Watch, Phase 5 pour GitHub)
- Diagramme "États de Phase" dans PhaseManager mis à jour (ARCHITECTURE → VALIDATION)
- **P1** Bug AAA : `VersionManager.create()` traite le cas post-ValidationPhase (meta.json existe, branche absente → crée seulement la branche)
- **P2** Bug BBB : `processAnswers()` matche `oui|non` + `_applyAnswer()` gère la branche oui/non pour `modified_task_file`
- **P2** Bug CCC : `DaemonHeartbeat.checkBriefing(projectRoot, io)` statique implémentée
- **P3** Bug DDD : Dead code FileSystem supprimé de `_onFileChanged()`
- **P3** Bug EEE : Déjà corrigé en v4 — confirmé ✅
- **P2** Bug A : `DaemonHeartbeat._generateBriefing()` — guard `if (total === 0) return` (anti division par zéro)
- **P2** Bug B : Handlers chokidar wrappés avec `.catch()` — plus d'UnhandledPromiseRejection
- **P2** Bug C : `BRANCH_MISMATCH` appelle `versionManager.switch()` au lieu d'afficher la commande git
- **P2** Bug D : `MANUAL_CHANGES` persiste réellement dans `progress.json` via `_persistManualChanges()` + actualise `lastSessionAt`
- **P3** Bug E : `ValidationPhase` ajoute `type: 'RELEASE'` dans le meta — cohérent avec `VersionManager.create()`
- **P1** Bug A : `SyncChecker.check()` retourne `activeVersion` dans `MANUAL_CHANGES` + `_persistManualChanges()` l'utilise correctement
- **P2** Bug B : `ArchitecturePhase` — les deux `JSON.parse` wrappés en try/catch avec messages explicites
- **P2** Bug A : Suppression du fallback `?? project.currentVersion` (ReferenceError si activeVersion est null)
**Arrêté à** : Fin de l'audit v8 — spécifications Phase 1-4 complètes, cohérentes, et sans bug connu. Prêt pour l'implémentation.
**Prochaine action** : Démarrer Phase 1 — Foundation — lire `.claude/docs/tasks/phase-1-foundation/INDEX.md`
