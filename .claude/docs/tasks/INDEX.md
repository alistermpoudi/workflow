# Workflow — Tâches de Build (Vue d'Ensemble)

## Stratégie

Build organisé autour de **7 piliers load-bearing** (voir `phase-0-vision/02-pillars.md`). L'ordre des phases découle de la nécessité technique, pas de l'arbitraire produit.

**Seuil critique** : fin de Phase 6. À ce point, Workflow Core est branchable sur Claude Code via MCP. **On l'utilise alors pour construire les phases 7-10** (dogfooding).

## Phases

| # | Phase | Objectif | Piliers couverts |
|---|-------|----------|------------------|
| 0 | [Vision & Architecture](phase-0-vision/) | Cadre conceptuel, 7 piliers, ordre de build | — |
| 1 | [Protocole & Persistence](phase-1-foundation/INDEX.md) | `.workflow/` versionné + skills + decisions-graph + contexts + sync | 1, 2 (foundation), 4, 7 |
| 2 | [Cerveau Cognitif](phase-2-cognitive/INDEX.md) | Multi-LLM par rôle + LLMContextLoader + CodePatcher chirurgical | 3, 5 |
| 3 | [Boucle d'Exécution](phase-3-execution-engine/INDEX.md) | ExecutionLoop + SkillCurator + TeachSystem + extensions | 2 (sources auto+teach), 5 |
| 4 | [Phases Projet (revisitables)](phase-4-project-phases/INDEX.md) | PhaseManager revisitable + Starter Mode + Discovery → Active | Support des piliers |
| 5 | [Agent & CLI](phase-5-agent-cli/INDEX.md) | WorkflowAgent + CLI + InFlowCorrector | Pilier 6 (prép) + 2 (in-flow) |
| 6 | [**MCP Server**](phase-6-mcp/INDEX.md) ⭐ | **Surface primaire — seuil dogfooding** | 6 (complet) |
| 7 | [Couches Proactives](phase-7-proactive/INDEX.md) | DaemonHeartbeat + WatchMode + ParallelExecutor | Support 2, 4 |
| 8 | [Surfaces Tierces](phase-8-surfaces/INDEX.md) | VS Code, GitHub, Telegram, REST API (clients MCP) | 6 (achevé) |
| 9 | [Robustesse](phase-9-robustness/INDEX.md) | CodeIndexer + WorkflowLibrary + BreakingChange + ProjectIngester | Renforce 1, étend 2 (ingestion) |
| 10 | [Polish & Différenciateurs](phase-10-polish/INDEX.md) | doc generate + audit + estimate + onboard | Met en valeur tous les piliers |

## Les 7 Piliers Load-Bearing

1. **Protocole `.workflow/`** — schéma versionné lisible par tous les clients
2. **Skills + Curator + 4 sources d'apprentissage** — `auto_retry` (Phase 3), `user_explicit` via `teach` (Phase 3), `in_flow_correction` via Ctrl+T (Phase 5), `project_ingestion` via `learn-from` (Phase 9). Niveau `USER_OVERRIDE` au-dessus de HIGH.
3. **Multi-LLM par rôle** — `reasoning` / `code_generation` / `fast` / `curator` / `compression`
4. **Décisions actives + graphe rétro-propageant** — détecte contradictions, propage aux tâches en cours
5. **CodePatcher chirurgical** — search/replace blocks dès le départ, jamais de fichier complet régénéré
6. **MCP Server comme surface primaire** — toutes les autres surfaces sont des clients du protocole
7. **Contexts spécialisés** — hiérarchie + héritage (`_global` → `mobile` → `mobile.flutter`), templates bundled, auto-detect, exportables. Workflow devient un **cabinet de devs spécialisés**, pas un agent à mémoire unique.

> Détail complet : [`phase-0-vision/02-pillars.md`](phase-0-vision/02-pillars.md)

## Anti-Patterns à Éviter

| Anti-pattern | Pourquoi c'est dangereux |
|---|---|
| "MVP minimal" qui sacrifie un pilier | Tue la promesse différenciante au profit de features cosmétiques |
| Régénérer fichiers entiers | Habitude technique médiocre difficile à défaire après |
| Single-LLM partout | Performance médiocre OU coût insoutenable |
| Phases waterfall figées | Anti-UX moderne — l'utilisateur doit pouvoir coder en 30s |
| `allowed_commands` whitelist gelée | Tue le flow dev — il faut apprentissage + mémoire |
| Intégrations sans protocole | N intégrations divergentes, chacune ré-implémente la logique |
| Skills sans Curator | Bruit accumulé, retour à un agent stateless dopé |
