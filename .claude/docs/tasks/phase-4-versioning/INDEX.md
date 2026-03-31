# Phase 4 — MCP Server (Workflow Core)

## Objectif

Exposer Workflow via MCP pour qu'il soit utilisable directement dans Claude Code, Cursor et tout agent compatible MCP. C'est la priorité stratégique : après cette phase, Workflow est utilisable dans son environnement naturel sans avoir à construire l'agent autonome complet.

## Dépendances

- Phase 3 complète ✅

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 4.1 | [01-mcp-server.md](01-mcp-server.md) | `MCPServer.js` — tous les outils MCP avec validation `allowed_commands` |
| 4.2 | [02-version-manager.md](02-version-manager.md) | `VersionManager.js` — cycle de vie versions + couplage Git complet |

## Critères de Sortie de Phase

- [ ] `workflow-mcp` peut être ajouté dans la config MCP de Claude Code
- [ ] Toutes les commandes de la liste d'outils MCP fonctionnent
- [ ] `allowed_commands` est validé avant toute exécution
- [ ] Les versions peuvent être créées, switchées, complétées et hotfixées
- [ ] Switch bloque si repo non propre
