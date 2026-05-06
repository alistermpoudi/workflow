# Phase 6 — MCP Server (Seuil Dogfooding)

## Objectif

Exposer Workflow via **MCP** comme **surface primaire** (Pilier 6). Toutes les autres surfaces (CLI, VS Code, Telegram, REST, GitHub) deviennent des **clients du protocole MCP** — pas des intégrations dupliquant la logique métier.

> **⭐ Seuil critique** : à la fin de cette phase, Workflow Core est branchable sur Claude Code, Cursor, ou tout client MCP. **On utilise alors Workflow pour construire les phases suivantes** — c'est le moment où la qualité du produit s'auto-vérifie par usage réel (dogfooding).

## Piliers Adressés

- **Pilier 6 (complet)** — MCP comme surface primaire
- Sécurité — `AllowedCommandsPolicy` avec apprentissage (Pilier 6, deuxième moitié)

## Stack Phase 6

```
mcp                  SDK Python officiel Anthropic (pip install mcp)
asyncio              Transport stdio
workflow.core        ProjectMemory, ContextManager, PhaseManager, etc.
allowed_commands.json Politique persistée + apprentissage
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 6.1 | [01-mcp-server.md](01-mcp-server.md) | `MCPServer.py` — serveur MCP avec ~20 outils (toute la surface Workflow) |
| 6.2 | [02-version-manager.md](02-version-manager.md) | `VersionManager.py` + `GitManager.py` complet — cycle de vie versions, branches, hotfix |
| 6.3 | [03-allowed-commands-policy.md](03-allowed-commands-policy.md) | **`AllowedCommandsPolicy.py`** — whitelist + apprentissage (3 niveaux : builtin, projet, prompt) |

## Dépendances

- Phases 1-5 complètes ✅

## Critères de Sortie de Phase

- [ ] `workflow-mcp` démarre et répond aux outils MCP en stdio
- [ ] Claude Code peut appeler `workflow_get_current_task()` et obtenir la tâche courante
- [ ] `workflow_version_switch()` bloque si le repo n'est pas propre (jamais de stash auto)
- [ ] `workflow_run_command()` consulte `AllowedCommandsPolicy.authorize()` avant exécution
- [ ] `AllowedCommandsPolicy.check('git status')` retourne ALLOW (builtin)
- [ ] `AllowedCommandsPolicy.check('rm -rf /')` retourne DENY (always_deny)
- [ ] `AllowedCommandsPolicy.check()` sur commande inconnue retourne ASK en interactif
- [ ] `AllowedCommandsPolicy._add_allowed(cmd, 'prefix')` extrait les 3 premiers tokens
- [ ] `VersionManager` crée une branche git `workflow/vX.Y` pour chaque version
- [ ] `VersionManager.hotfix(name, reason)` crée `workflow/hotfix/X.Y.Z` depuis la branche parent
- [ ] **Dogfooding démarré** : on installe Workflow MCP dans Claude Code et on s'en sert pour tâches Phase 7+

## Note Stratégique

C'est ici que la valeur réelle se vérifie. Si Workflow MCP est inutilisable au quotidien, **ne pas avancer en Phase 7** — investiguer pourquoi avant d'ajouter de nouvelles couches.
