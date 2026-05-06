# Phase 0 — Les 7 Piliers Load-Bearing

## À quoi sert ce document

Ce document explicite les **7 bets architecturaux** sur lesquels Workflow repose. Si l'un d'entre eux est mal posé, l'agent ne peut pas tenir sa promesse différenciante. Tout le reste — daemons, intégrations, surfaces tierces — est construit *autour* de ces piliers, jamais à leur place.

L'ordre de build des phases (1 à 10) découle directement de ces piliers : les fondations cognitives d'abord, le dogfooding via MCP au plus tôt, les couches accessoires après.

---

## Pilier 1 — `.workflow/` comme protocole formalisé

**Bet** : `.workflow/` n'est pas un dossier de fichiers ad-hoc, c'est un **schéma versionné** avec contrats de lecture/écriture, compatibilité ascendante, et spec lisible par des outils tiers.

**Pourquoi load-bearing** : VS Code extension, Telegram bot, GitHub integration, REST API, web dashboard — tous ces "interfaces" deviennent des **clients du protocole**, pas des intégrations à réécrire à la main. Sans cette discipline, on construit N intégrations qui finissent par diverger.

**Implications concrètes** :
- Champ `schema_version` dans `project.json`
- Migration automatique d'une version de schéma à la suivante
- Validation par JSON Schema au boot (`SyncChecker` refuse de continuer si le schéma est cassé)
- Documentation publique du protocole — n'importe qui peut écrire un client compatible

**Tâche dédiée** : `phase-1-foundation/00-protocol.md`

---

## Pilier 2 — Apprentissage cumulé : 4 sources, Curator, Réutilisation

**Bet** : Le vrai différenciateur de Workflow n'est pas la mémoire de session — c'est la **mémoire institutionnelle cross-projet**. Workflow accumule des skills à travers **4 sources distinctes** ; un Curator LLM consolide périodiquement pour dédupliquer, promouvoir au global, supprimer le bruit.

### Les 4 sources d'apprentissage

| Source | Trigger | Confidence | Quand |
|---|---|---|---|
| `auto_retry` | retry réussi dans ExecutionLoop | `MEDIUM` | Phase 3 |
| `user_explicit` | `workflow teach` (ou `/teach` MCP) | `USER_OVERRIDE` | Phase 3 |
| `user_negative` | `workflow avoid` (anti-patterns) | `USER_OVERRIDE` | Phase 3 |
| `project_ingestion` | `workflow learn-from <projet>` | `HIGH` | Phase 9 |
| `in_flow_correction` | Dev interrompt l'agent (Ctrl+T) | `USER_OVERRIDE` | Phase 5 |

**Le niveau `USER_OVERRIDE`** est nouveau : il bat tout (HIGH-LLM-inferred, MEDIUM, contradictions auto-détectées). Le dev a raison contre l'agent par défaut.

**Pourquoi load-bearing** : C'est le seul mécanisme qui rend l'agent *plus intelligent à chaque projet*. Sans Curator, on accumule du bruit ; sans skills, on retombe à un agent stateless dopé. Sans `user_explicit` / `learn-from`, on attend des mois que l'agent découvre par échec ce que le dev sait déjà. Cette boucle 4-sources est la traduction technique de "agent qui apprend comme un humain freelance".

**Implications concrètes** :
- `SkillManager` (Phase 1) — CRUD de skills, **context-aware** (voir Pilier 7)
- `ExecutionLoop` (Phase 3) — création auto skill après retry réussi
- `TeachSystem` (Phase 3) — `workflow teach` / `workflow avoid` + `USER_OVERRIDE`
- `SkillCurator` (Phase 3) — consolidation périodique + cross-context promotion
- `InFlowCorrector` (Phase 5) — Ctrl+T interrupt → skill USER_OVERRIDE → retry
- `ProjectIngester` (Phase 9) — `workflow learn-from <path>` avec tags + review
- `LLMContextLoader` (Phase 2) — injection des skills pertinents dans chaque prompt
- Granularité : un skill ≠ un fix individuel ; un skill = un *pattern*

**Tâches dédiées** :
- `phase-1-foundation/05-skill-system.md`
- `phase-3-execution-engine/02-skill-curator.md`
- `phase-3-execution-engine/04-teach-system.md`
- `phase-5-agent-cli/03-in-flow-correction.md`
- `phase-9-robustness/04-project-ingestion.md`

---

## Pilier 3 — Multi-LLM par rôle

**Bet** : Le LLM qui génère 50 tâches détaillées (`reasoning`) n'est pas le même que celui qui code (`code_generation`), ni que celui qui score 200 décisions (`fast`), ni que celui qui consolide les skills (`curator`), ni que celui qui résume une session (`compression`). Chaque rôle a ses optimums coût/qualité/latence.

**Pourquoi load-bearing** : C'est ce qui permet à Workflow d'être à la fois (1) précis sur les phases cognitives profondes, (2) rapide sur les opérations de masse, (3) abordable sur les phases répétitives. Sans cette architecture, soit on paye Opus partout (insoutenable), soit on tape Haiku partout (médiocre).

**Implications concrètes** :
- `LLMProvider` avec routage par rôle via `workflow.config.yaml`
- Chaque appel LLM doit déclarer son rôle (pas de défaut implicite)
- Fallback sur `default_model` si le modèle spécialisé est indisponible
- Configuration 100% locale possible (Ollama) pour offline

**Tâche dédiée** : `phase-2-cognitive/01-llm-provider.md`

---

## Pilier 4 — Decisions actives + graphe rétro-propageant

**Bet** : Le `decisions.log` n'est pas un journal — c'est une **base de connaissance technique active**. Chaque décision est un nœud dans un graphe avec relations (`CONTRADICTS`, `DEPENDS_ON`, `SUPERSEDES`, `REFINES`). Quand une décision change, Workflow détecte automatiquement les tâches *en cours* impactées et les annote.

**Pourquoi load-bearing** : C'est ce qui transforme Workflow d'un "exécutant qui suit des décisions" en "PM technique qui les fait respecter cohéremment dans le temps". Aucun outil actuel ne fait ça. Combiné aux skills, c'est ce qui justifie le terme *agent qui réfléchit comme un humain*.

**Implications concrètes** :
- `DecisionsLog` écrit en texte brut (lisible humain) ET indexe en SQLite FTS5
- `DecisionsGraph` stocke relations + niveaux de confiance (HIGH/MEDIUM/LOW)
- À chaque ajout d'une décision : check des contradictions avec les existantes
- Si nouvelle décision contredit une ancienne → alerte + propagation aux tâches `pending`
- `ContextManager._load_scored_decisions()` utilise le graphe pour le scoring

**Tâches dédiées** :
- `phase-1-foundation/06-decisions-log.md`
- `phase-1-foundation/07-decisions-graph.md`

---

## Pilier 5 — CodePatcher chirurgical dès le départ

**Bet** : Workflow ne régénère **jamais** un fichier entier pour appliquer une modification. Edits via search/replace blocks, AST patches (tree-sitter), ou tool-based file edits. Un fichier de 500 lignes qui change de 10 lignes = un patch de 10 lignes.

**Pourquoi load-bearing** : Régénérer des fichiers entiers est techniquement médiocre — coût tokens output × 10, hallucinations sur les portions inchangées, latence absurde. Tous les agents Hermes-tier (Aider, Cursor, Claude Code) le font depuis 2024. Le différer en v1.5 condamne le MVP à être lent et inexact.

**Implications concrètes** :
- `CodePatcher` (Phase 2) — search/replace blocks par défaut, fallback AST si ambigu
- `ExecutionLoop` génère des *patches*, pas des fichiers complets
- `PromptBuilder.generate_code()` instruit le LLM en format patch
- Tree-sitter integré dès le début (pas v1.5)

**Tâche dédiée** : `phase-2-cognitive/04-code-patcher.md`

---

## Pilier 6 — MCP Server comme surface primaire

**Bet** : MCP est la surface d'interaction **principale** de Workflow Core. CLI, VS Code extension, Telegram bot, REST API, GitHub integration — tous des **clients du même protocole MCP**. Pas N intégrations séparées qui ré-implémentent la même logique.

**Pourquoi load-bearing** : Cette discipline empêche l'explosion en surfaces incohérentes. Elle permet aussi le dogfooding immédiat (Workflow utilisable depuis Claude Code dès Phase 6) avant même que l'agent autonome existe. C'est le chemin le plus court vers la valeur.

**Implications concrètes** :
- Phase 6 = seuil de **dogfooding** — après ça, Workflow construit Workflow
- `MCPServer` expose tous les outils (~20) avec validation `allowed_commands`
- VS Code extension = client MCP, pas couche custom
- Telegram bot = client MCP wrapping
- `allowed_commands` policy avec **apprentissage** (whitelist + prompt + mémoire), pas figée

**Tâches dédiées** :
- `phase-6-mcp/01-mcp-server.md`
- `phase-6-mcp/03-allowed-commands-policy.md`

---

## Pilier 7 — Contexts : expertises spécialisées

**Bet** : Un dev senior n'a pas *une* mémoire indifférenciée — il a des **mental models distincts par domaine** (mobile ≠ web ≠ backend ≠ data). Workflow réplique ça via un système de **contexts** hiérarchiques avec héritage : `_global`, `mobile`, `mobile.flutter`, `mobile.react-native`, `web.nextjs`, etc. Chaque context a ses propres skills, decisions, defaults, design preferences.

**Pourquoi load-bearing** : Sans contexts, après 5 projets mobile + 5 projets web, les skills `auth` se polluent entre eux (JWT mobile ≠ NextAuth.js). Le pool global devient bruit. Avec contexts, c'est **un cabinet de devs spécialisés** que le freelance peut activer selon le projet. Et c'est ce qui transforme la thèse "agent qui apprend à travers tes projets" en "**agent qui développe des expertises spécialisées dans tes domaines**".

**Comportement clé** :
- Un context **n'existe que s'il est utilisé** (lazy instantiation depuis templates bundled)
- Au `workflow init`, **auto-détection** depuis les fichiers du projet (pubspec.yaml → mobile.flutter, package.json+next → web.nextjs)
- **Multi-contexts actifs** : un projet peut activer `[_global, web, web.nextjs]` simultanément
- **Héritage** : `mobile.flutter` hérite de `mobile` qui hérite de `_global`. Skills/decisions résolus du plus spécifique au plus général
- **Cross-context promotion** : Curator détecte les patterns présents dans 2+ contexts → propose promotion au parent commun
- **Templates portables** : `workflow context export mobile.flutter` → partagé sur `workflow-hub`

**Implications concrètes** :
- `ContextRegistry` (Phase 1) — CRUD des contexts, hiérarchie, héritage
- Templates bundled dans le package Workflow (`mobile.flutter`, `web.nextjs`, etc. — sans skills, juste config)
- `project.json#active_contexts: [string]` — liste des contexts actifs pour ce projet
- `SkillManager`, `DecisionsLog`, `DecisionsGraph` deviennent **context-aware**
- `LLMContextLoader` (renommé depuis `ContextManager` pour éviter la collision) charge skills/decisions des contexts actifs avec poids par spécificité
- `workflow-hub` (Phase 8) prend tout son sens — pas juste des patterns épars, mais des **expertises packagées**

**Tâche dédiée** : `phase-1-foundation/09-contexts.md`

> **Naming critical** : on a renommé `ContextManager` → `LLMContextLoader` (Phase 2) pour libérer le mot "context" pour le concept utilisateur. C'est ce mot que le dev utilise — on ne va pas inventer un jargon.

---

## Ordre de Build Justifié

```
Phase 0  Vision & Architecture          ─── Cadre conceptuel
Phase 1  Protocole & Persistence        ─── Piliers 1 + 4 + 7 (foundations + contexts)
Phase 2  Cerveau Cognitif               ─── Piliers 3 + 5
Phase 3  Boucle d'Exécution             ─── Pilier 2 (skills 4 sources + curator + teach)
Phase 4  Phases Projet (revisitables)   ─── Le workflow utilisateur
Phase 5  Agent & CLI                    ─── Surface humaine + in-flow correction
Phase 6  MCP Server                     ─── Pilier 6 ◀── DOGFOODING START
Phase 7  Couches Proactives             ─── Daemon, Watch, Parallel
Phase 8  Surfaces Tierces               ─── VS Code, Telegram, GitHub, REST
Phase 9  Robustesse                     ─── Indexer, Library, Breaking, Project Ingestion
Phase 10 Polish & Différenciateurs      ─── doc gen, audit, estimate, onboard
```

**Le seuil clé** : fin de Phase 6. À ce point, Workflow Core est branchable sur Claude Code via MCP, et **on l'utilise pour construire les phases suivantes**. Les phases 7-10 sont prioritisées par l'usage réel, pas par la spec figée.

---

## Anti-patterns à éviter

| Anti-pattern | Pourquoi c'est dangereux |
|---|---|
| "Faisons un MVP minimal d'abord" | Tue les piliers load-bearing au profit de features cosmétiques |
| Régénérer fichiers entiers en attendant CodePatcher | Condamne le MVP à être lent/inexact, habitudes prises difficiles à défaire |
| Single-LLM partout | Performance médiocre OU coût insoutenable selon le choix de modèle |
| Phases waterfall figées | UX anti-moderne — l'utilisateur doit pouvoir coder en 30s |
| `allowed_commands` whitelist gelée | Tue le flow dev — il faut apprentissage + mémoire |
| Intégrations sans protocole | N intégrations divergentes, chacune ré-implémente la logique |
| Skills sans Curator | Bruit accumulé, retour à un agent stateless dopé |

---

## Référence pour les futures décisions

Quand une nouvelle feature est proposée pour Workflow, la question à se poser :

> **Sert-elle un des 7 piliers, ou s'y branche-t-elle proprement ?**

Si oui : OK, place-la dans la bonne phase.
Si non : c'est probablement un nice-to-have qui appartient à une roadmap secondaire ou à reporter à l'après-dogfooding.
