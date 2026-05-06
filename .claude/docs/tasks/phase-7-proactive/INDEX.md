# Phase 7 — Couches Proactives

## Objectif

Ajouter les **couches proactives** qui rendent Workflow présent en arrière-plan sans interrompre l'utilisateur :

- `DaemonHeartbeat` — surveillance continue + briefing matinal + lance le `SkillCurator` automatiquement
- `WatchMode` — annotation passive des modifications hors scope
- `ParallelExecutor` — exécution en parallèle des tâches indépendantes via `git worktrees`

> **Place après MCP (Phase 6)** : on commence par avoir un produit utilisable en sync mode propre, ensuite on ajoute le proactif. Sinon, on debug du proactif avant que la base soit solide.

## Piliers Adressés

- Support **Pilier 2** — Daemon lance Curator automatiquement (≥10 nouveaux skills)
- Support **Pilier 4** — Daemon alerte sur les contradictions de décisions

## Stack Phase 7

```
asyncio              Boucle async
watchfiles           Surveillance fichiers (équivalent chokidar)
git worktree         Isolation pour exécution parallèle
launchd / systemd    Daemon en arrière-plan (par OS)
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 7.1 | [01-daemon-heartbeat.md](01-daemon-heartbeat.md) | `DaemonHeartbeat.py` — daemon proactif (briefing + surveillance + auto-curator) |
| 7.2 | [02-watch-mode.md](02-watch-mode.md) | `WatchMode.py` — annotation passive via `watchfiles.awatch` → `.workflow/questions/` |
| 7.3 | [03-parallel-executor.md](03-parallel-executor.md) | `ParallelExecutor.py` — exécution parallèle via git worktrees (tâches indépendantes) |

## Dépendances

- Phases 1-6 complètes ✅
- **Dogfooding** Phase 6 actif — on utilise Workflow MCP pour construire ces tâches

## Critères de Sortie de Phase

- [ ] `DaemonHeartbeat` tourne en arrière-plan (sans bloquer l'utilisateur)
- [ ] Briefing quotidien généré dans `.workflow/briefings/YYYY-MM-DD.md`
- [ ] `DaemonHeartbeat.maybe_run_curator()` lance le Curator si ≥10 nouveaux skills
- [ ] `DaemonHeartbeat` alerte sur les contradictions actives détectées par `DecisionsGraph`
- [ ] `WatchMode` crée des fichiers questions dans `.workflow/questions/` quand un fichier hors scope est modifié
- [ ] `WatchMode.process_answers()` ingère les réponses utilisateur au prochain `workflow start`
- [ ] `ParallelExecutor` exécute les tâches indépendantes en parallèle via git worktrees
- [ ] `ParallelExecutor` merge les branches dans la branche version après succès
- [ ] `ParallelExecutor` détecte les conflits avant exécution (graphe de dépendances)
- [ ] Aucune interruption visible de l'utilisateur pendant qu'il code en parallèle
