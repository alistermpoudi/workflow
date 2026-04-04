# Phase 5 — Tâche 5.2 : GitHubIntegration.js

## Objectif

Connecter Workflow aux événements GitHub/GitLab. Quand une PR est mergée, la tâche correspondante est automatiquement marquée DONE. Quand un CI casse, le daemon reçoit l'information et prépare une analyse.

## Dépendances

- Phase 4 (MCP Server) ✅
- Tâche 3.4 (DaemonHeartbeat) ✅

## Fichiers à Créer

- `src/interfaces/GitHubIntegration.js` [CRÉER]
- `tests/unit/GitHubIntegration.test.js` [CRÉER]

## Événements Gérés

| Événement GitHub | Action Workflow |
|-----------------|-----------------|
| PR mergée | Cherche la tâche liée (par titre/branche) → marque DONE |
| CI failed | Notifie le daemon → analyse de l'erreur |
| Issue créée avec label `workflow` | Crée une tâche dans la version active |
| Release publiée | Marque la version comme COMPLETED |

## Liaison PR ↔ Tâche

La liaison se fait par convention de nommage :
- Branche : `workflow/v1.0/TASK-007` → lié à TASK-007
- Titre PR : `[TASK-007] API routes client` → lié à TASK-007
- Corps PR : `Closes TASK-007` → lié à TASK-007

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | PR mergée avec branche `workflow/*/TASK-XXX` → tâche marquée DONE | ⬜ |
| 2 | CI failed → fichier créé dans `.workflow/alerts/` avec l'erreur | ⬜ |
| 3 | La liaison fonctionne sur GitHub ET GitLab | ⬜ |
| 4 | Si aucune tâche trouvée pour une PR → log silencieux, pas d'erreur | ⬜ |
