# Phase 3 — Exécution de Base

## Objectif

Implémenter `ExecutionLoop.js` (boucle build/validate/correct) et la CLI readline minimale. À l'issue de cette phase, Workflow peut exécuter des tâches, valider avec la commande définie dans `tech-stack.json`, s'auto-corriger 3 fois, et escalader à l'utilisateur.

## Dépendances

- Phase 2 complète ✅

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 3.1 | [01-execution-loop.md](01-execution-loop.md) | `ExecutionLoop.js` — build_validate + 3 retries + escalade |
| 3.2 | [02-cli.md](02-cli.md) | `CLI.js` — interface readline + chalk, commandes de base |
| 3.3 | [03-workflow-agent.md](03-workflow-agent.md) | `WorkflowAgent.js` — orchestrateur principal (boucle session) |

## Critères de Sortie de Phase

- [ ] `workflow start` démarre une session et conduit les 5 phases
- [ ] `workflow run` exécute la prochaine tâche en attente
- [ ] L'auto-correction fonctionne sur 3 tentatives max
- [ ] L'escalade à l'utilisateur affiche le contexte d'erreur complet
- [ ] `SyncChecker` est appelé au démarrage de chaque session
