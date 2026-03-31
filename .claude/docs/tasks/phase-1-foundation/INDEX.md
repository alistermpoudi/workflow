# Phase 1 — Foundation

## Objectif

Construire la couche de persistance et de synchronisation qui est le coeur de Workflow. À l'issue de cette phase, Workflow peut lire et écrire tous les fichiers `.workflow/`, détecter le state drift entre sessions, gérer les tâches, et enregistrer/consulter les décisions techniques.

**Aucune IA, aucun LLM** dans cette phase — uniquement des opérations fichiers, Git et JSON.

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 1.1 | [01-monorepo-init.md](01-monorepo-init.md) | Structure du projet, `package.json`, ESLint, Vitest |
| 1.2 | [02-filesystem.md](02-filesystem.md) | `FileSystem.js` — opérations fichiers sur `.workflow/` |
| 1.3 | [03-project-memory.md](03-project-memory.md) | `ProjectMemory.js` — CRUD `project.json`, `vision.md`, `features.json`, `tech-stack.json` |
| 1.4 | [04-task-manager.md](04-task-manager.md) | `TaskManager.js` — CRUD `TASK-XXX.md` + `progress.json` |
| 1.5 | [05-decisions-log.md](05-decisions-log.md) | `DecisionsLog.js` — écriture et lecture active |
| 1.6 | [06-sync-checker.md](06-sync-checker.md) | `SyncChecker.js` — state drift + vérification branche Git |

## Dépendances

Aucune dépendance externe — cette phase peut démarrer immédiatement.

## Critères de Sortie de Phase

- [ ] `npm test` passe avec 100% de tests unitaires sur les 6 modules
- [ ] On peut créer un projet `.workflow/` complet depuis zéro
- [ ] On peut lire l'état d'un projet `.workflow/` existant
- [ ] `SyncChecker` détecte correctement un state drift (branche incorrecte, fichiers modifiés)
- [ ] `DecisionsLog` enregistre et relit les décisions correctement
