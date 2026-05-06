# Phase 1 — Protocole & Persistence

## Objectif

Construire la **fondation du protocole `.workflow/`** + la couche de persistance qui implémente ce protocole. À l'issue de cette phase, Workflow peut lire et écrire tous les fichiers `.workflow/` selon un schéma versionné, détecter le state drift, gérer les tâches, accumuler des skills cross-projet, et entretenir un graphe actif de décisions techniques avec détection de contradictions.

**Aucune IA, aucun LLM** dans cette phase — uniquement opérations fichiers, Git, SQLite, YAML, JSON Schema. Le LLM apparaît en Phase 2.

## Piliers Adressés

- **Pilier 1** — `.workflow/` comme protocole versionné (tâche 1.0)
- **Pilier 2** — Skills cross-projet : couche de persistance (tâche 1.5 ; les sources actives — auto_retry, teach, ingestion, in-flow — viennent en Phases 3/5/9)
- **Pilier 4** — Décisions actives + graphe rétro-propageant (tâches 1.6 + 1.7)
- **Pilier 7** — Contexts hiérarchiques + héritage + templates (tâche 1.9)

## Stack Phase 1

```
Python 3.12+         uv sync --extra dev
SQLite               aiosqlite (decisions.db avec FTS5)
JSON Schema          jsonschema (validation protocole)
Fichiers             aiofiles + pathlib
Git                  asyncio.create_subprocess_exec
YAML                 pyyaml (skills frontmatter)
Tests                pytest + pytest-asyncio
Lint                 ruff
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 1.0 | [00-protocol.md](00-protocol.md) | **Protocole `.workflow/`** — schéma versionné, JSON Schemas, migrations, spec publique |
| 1.1 | [01-monorepo-init.md](01-monorepo-init.md) | Structure du projet, `pyproject.toml`, Ruff, Pytest |
| 1.2 | [02-filesystem.md](02-filesystem.md) | `FileSystem.py` — opérations fichiers async + atomic write |
| 1.3 | [03-project-memory.md](03-project-memory.md) | `ProjectMemory.py` — reference impl du protocole (CRUD `project.json`, `vision.md`, `features.json`, `tech-stack.json`) |
| 1.4 | [04-task-manager.md](04-task-manager.md) | `TaskManager.py` — CRUD `TASK-XXX.md` + `progress.json` (avec `Intent`, `scope`, `tags`) |
| 1.5 | [05-skill-system.md](05-skill-system.md) | `SkillManager.py` — système de skills cross-projet (CRUD + recherche) |
| 1.6 | [06-decisions-log.md](06-decisions-log.md) | `DecisionsLog.py` — SQLite FTS5, écriture + lecture active |
| 1.7 | [07-decisions-graph.md](07-decisions-graph.md) | **`DecisionsGraph.py`** — graphe + relations + détection contradictions + rétro-propagation |
| 1.8 | [08-sync-checker.md](08-sync-checker.md) | `SyncChecker.py` — state drift + branche Git + validation protocole + migration auto |
| 1.9 | [09-contexts.md](09-contexts.md) | **`ContextRegistry.py`** — contexts hiérarchiques + héritage + templates bundled + auto-detect (Pilier 7) |

## Dépendances

Aucune dépendance externe — cette phase peut démarrer immédiatement après la lecture de Phase 0.

## Critères de Sortie de Phase

- [ ] `uv run pytest` passe avec 100% de tests unitaires sur les 9 modules
- [ ] On peut créer un projet `.workflow/` complet depuis zéro
- [ ] On peut lire l'état d'un projet `.workflow/` existant
- [ ] `ProtocolValidator` rejette les fichiers `.workflow/` malformés avec messages clairs
- [ ] Une migration de `schema_version` (mock v1.0.0 → v1.1.0) s'applique correctement
- [ ] `SyncChecker` détecte un state drift (branche incorrecte, fichiers modifiés)
- [ ] `DecisionsLog` enregistre et recherche les décisions via SQLite FTS5
- [ ] `DecisionsGraph` détecte automatiquement une contradiction entre 2 décisions sur même scope
- [ ] `DecisionsGraph.propagate_change()` annote le Journal des tâches `pending` impactées
- [ ] `SkillManager` crée et recherche des skills locaux et globaux, **context-aware**
- [ ] `ContextRegistry` instantie `_global` au boot, lit les templates bundled
- [ ] `ContextRegistry.create_from_template('mobile.flutter')` crée mobile + mobile.flutter
- [ ] `ContextRegistry.auto_detect()` propose le bon context depuis pubspec.yaml/package.json/etc.
- [ ] `ContextRegistry.export()` / `install()` permettent partage de contexts via archives
- [ ] `docs/PROTOCOL.md` documente la totalité de la structure `.workflow/` + contexts
