# Phase 6 — Robustesse (V1.5)

## Objectif

Rendre Workflow robuste sur de vrais projets de taille moyenne à grande. Les outils de cette phase ne sont pas nécessaires pour le MVP mais font la différence sur des projets de plus de 20 fichiers.

## Dépendances

- Phase 4 (MVP) complète ✅

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 6.1 | [01-code-patcher.md](01-code-patcher.md) | `CodePatcher.js` — diffs chirurgicaux + fallback AST tree-sitter |
| 6.2 | [02-code-indexer.md](02-code-indexer.md) | `CodeIndexer.js` — index JSON + variantes LLM + ripgrep |
| 6.3 | [03-context-manager-advanced.md](03-context-manager-advanced.md) | Chargement sélectif avancé dans `ContextManager` |

## Notes sur `CodePatcher`

En MVP, `ExecutionLoop` écrit les fichiers en entier (via `applyFiles`). `CodePatcher` apporte une alternative pour les fichiers volumineux — remplacer une fonction précise sans réécrire le fichier. Le fallback AST via `tree-sitter` est activé quand le ciblage par texte échoue.

## Notes sur `CodeIndexer`

En MVP, `ContextManager.loadOnDemand` est non-implémenté. `CodeIndexer` construit et maintient un index JSON des fonctions/classes/exports et utilise `ripgrep` en subprocess pour la recherche. La requête LLM génère des variantes de noms pour augmenter le rappel.

```javascript
// Exemple d'utilisation
const variants = ['formatDate', 'parseDate', 'dateUtils', 'timestampToIso'];
const match = await CodeIndexer.searchByVariants(variants);
// → trouve formatDate() dans src/utils/date.js
```
