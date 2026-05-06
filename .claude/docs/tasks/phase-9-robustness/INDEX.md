# Phase 9 — Robustesse

## Objectif

Renforcer Workflow pour les projets de taille moyenne à grande (200+ fichiers, 100+ tâches sur plusieurs versions). Indexation sémantique, capitalisation cross-projet, détection des breaking changes avant qu'ils explosent en production.

## Piliers Adressés

- Renforce **Pilier 1** — index permet de scaler le protocole sur gros projets
- Étend **Pilier 2** — `WorkflowLibrary` cross-projet (au-delà des skills individuels)

## Stack Phase 9

```
tree-sitter          AST parsing (déjà installé Phase 2)
tree-sitter-languages Grammaires multi-langage
ripgrep (subprocess) Recherche full-text rapide
hashlib              SHA256 pour invalidation d'index
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 9.1 | [01-code-indexer.md](01-code-indexer.md) | **`CodeIndexer.py`** — index symboles + imports via tree-sitter, mise à jour incrémentale + ripgrep |
| 9.2 | [02-workflow-library.md](02-workflow-library.md) | `WorkflowLibrary.py` — patterns cross-projet (consolidation skills entre projets) |
| 9.3 | [03-breaking-change-detector.md](03-breaking-change-detector.md) | `BreakingChangeDetector.py` — détecte les ruptures d'API/contrats avant merge |
| 9.4 | [04-project-ingestion.md](04-project-ingestion.md) | **`ProjectIngester.py`** — `workflow learn-from <projet>` avec tags + review (Pilier 2 — source `project_ingestion`) |

## Dépendances

- Phases 1-8 complètes ✅
- Dogfooding actif depuis Phase 6 — alimente les patterns de la `WorkflowLibrary`

## Critères de Sortie de Phase

- [ ] `CodeIndexer.rebuild()` indexe un projet de 1000 fichiers en < 30s
- [ ] `CodeIndexer.update_file()` met à jour un seul fichier sans relire tout
- [ ] `CodeIndexer.query('login')` retourne les locations exactes du symbole
- [ ] `CodeIndexer.grep(pattern)` utilise ripgrep et retourne les locations
- [ ] `ContextManager.loadOnDemand()` consomme `CodeIndexer.query()` (Phase 2 mise à jour)
- [ ] `WatchMode.on_file_change()` met à jour l'index incrémentalement
- [ ] `WorkflowLibrary` partage les skills réutilisables entre projets de l'utilisateur
- [ ] `WorkflowLibrary` propose des patterns au démarrage d'un nouveau projet similaire
- [ ] `BreakingChangeDetector` détecte une signature de fonction modifiée
- [ ] `BreakingChangeDetector` détecte un export retiré
- [ ] `BreakingChangeDetector` alerte l'utilisateur AVANT le merge avec impact estimé
- [ ] `workflow learn-from <path> --context X --learn architecture,patterns` produit un brouillon de skills
- [ ] `ProjectIngester` n'écrit JAMAIS dans le projet source (analyse read-only)
- [ ] Chaque skill proposé est validé un par un (pas d'absorption silencieuse)
- [ ] `--dry-run` montre les proposals sans commit
- [ ] Les skills ingérés ont `source='project_ingestion'`, `confidence='HIGH'`, tag de catégorie
- [ ] `~/.workflow/ingestions.log` enregistre l'audit trail
