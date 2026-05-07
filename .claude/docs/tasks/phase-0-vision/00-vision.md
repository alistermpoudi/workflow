# Phase 0 — Vision Produit

## Le Problème

Les agents de code actuels souffrent d'un défaut structurel : ils vivent dans le contexte de conversation. Quand ce contexte sature, l'agent perd le fil. L'utilisateur recommence tout depuis zéro dans une nouvelle session. Pire : ces agents ne comprennent jamais un projet dans sa globalité — on leur soumet des tâches isolées sans vision d'ensemble. Et même quand un agent résout brillamment un problème, **il oublie tout au projet suivant** : aucune capitalisation d'expérience.

## La Solution : Workflow

**Workflow** est un agent de code pour freelance et développeur indépendant qui **se souvient et accumule de l'expérience**, comme un humain qui a fait 30 projets et applique ses leçons.

Deux mémoires distinctes, persistantes :

1. **Mémoire projet** (`.workflow/` dans le projet) — vision, fonctionnalités, stack, décisions, état d'avancement. Workflow reprend exactement où il s'est arrêté à chaque session.
2. **Mémoire institutionnelle** (`~/.workflow/skills/` global) — patterns appris à travers tous les projets. Chaque retry réussi devient un skill ; un Curator LLM consolide périodiquement. **Workflow devient plus intelligent à chaque projet.**

> **Thèse différenciatrice** : Cursor / Claude Code sont des "assistants experts dans la session". Workflow est un **PM technique avec mémoire institutionnelle**. Quand tu démarres un projet React, il sait déjà comment tu structures tes hooks, quelles erreurs Prisma reviennent toujours, et quels choix d'architecture tu as faits sur les 5 derniers projets similaires.

### Les 7 Piliers Load-Bearing

1. **Protocole `.workflow/`** — schéma versionné lisible par tous les clients (CLI, VS Code, Telegram, web). Pas un dossier ad-hoc.
2. **Skills + Curator + 4 sources d'apprentissage** — `auto_retry` (échec→succès), `user_explicit` (`workflow teach`), `in_flow_correction` (Ctrl+T), `project_ingestion` (`learn-from`). Niveau `USER_OVERRIDE` au-dessus de HIGH.
3. **Multi-LLM par rôle** — `reasoning` pour penser, `code_generation` pour coder, `fast` pour scorer, `curator` pour consolider. Chaque rôle son optimum.
4. **Décisions actives + graphe rétro-propageant** — `decisions.log` + graphe de relations (CONTRADICTS, DEPENDS_ON, SUPERSEDES). Détecte les contradictions et propage aux tâches en cours.
5. **CodePatcher chirurgical** — diffs et search/replace blocks dès le départ. Jamais de régénération de fichier entier.
6. **MCP Server comme surface primaire** — toutes les autres surfaces (CLI, VS Code, Telegram, REST) sont des clients du protocole MCP.
7. **Contexts spécialisés** — hiérarchie `_global → mobile → mobile.flutter`, templates bundled, héritage, exportables. Workflow devient un **cabinet de devs spécialisés**, pas un agent à mémoire unique.

> Détail complet : voir `02-pillars.md`.

---

## Deux Modes d'Existence

### Workflow Agent
Agent autonome complet. L'utilisateur interagit avec lui comme avec un développeur-réalisateur. Workflow est à la fois le LLM (via Claude API) et le gestionnaire de projet. Disponible en CLI ou via Telegram.

### Workflow Core
Workflow comme gestionnaire de projet pur, intégrable dans d'autres agents via MCP. Dans ce mode, Workflow n'est **pas** le LLM — il est la mémoire, la structure et les outils. Le modèle qui réfléchit, c'est l'agent hôte (Claude Code, Cursor, etc.).

> **Stratégie** : Construire Workflow Core en premier. C'est le chemin le plus court vers quelque chose d'utilisable — branché sur Claude Code en quelques semaines.

---

## Les 4 Sources d'Apprentissage

Workflow n'apprend pas seulement par échec. Il a **4 voies d'apprentissage cumulatives**, du passif au plus actif :

### 1. `auto_retry` — apprentissage par échec→succès (passif)
Quand `ExecutionLoop` corrige une erreur après retry, le pattern devient un skill (confidence: `MEDIUM`). C'est l'apprentissage le plus lent mais entièrement automatique.

### 2. `user_explicit` — `workflow teach` / `workflow avoid` (actif)
```bash
workflow teach "go_router toujours pour la nav, jamais Navigator 1.0"
workflow teach --context mobile.flutter "j'utilise toujours BLoC, pas Provider"
workflow avoid "ne propose plus de class-based React components"
workflow avoid --personal "pas d'emoji dans les commits"
```
Le dev sait ce qu'il veut. Skill créé avec `confidence: USER_OVERRIDE` (le plus haut). Bat les patterns auto-créés.

### 3. `in_flow_correction` — Ctrl+T pendant l'agent (correctif)
```
[Workflow génère du code médiocre]
[Dev appuie Ctrl+T]
✋ Que dois-je corriger ?
> utilise FastAPI Depends, pas @middleware
✓ Skill USER_OVERRIDE créé. Retry avec la correction injectée.
```
La correction la plus précieuse — le contexte du moment est riche. Tâche en cours retentée immédiatement.

### 4. `project_ingestion` — `workflow learn-from` (massif)
```bash
workflow learn-from ~/projects/old-react-app \
  --context web.nextjs \
  --learn architecture,naming,patterns \
  --ignore styling,deps
```
Le dev pointe un ancien projet, tag ce qu'il veut absorber. Workflow analyse, propose des skills (review un par un), et les ingère dans le context cible. Capitalise des années de travail passé.

> Le `SkillCurator` (LLM `role='curator'`) consolide périodiquement : déduplique, promeut au global les patterns confirmés, archive les skills jamais utilisés. Les skills `USER_OVERRIDE` ne sont jamais archivés sans confirmation.

---

## Les Contexts — Cabinet de Devs Spécialisés (Pilier 7)

Un dev senior n'a pas *une* mémoire indifférenciée — il a des **mental models distincts** par domaine. Workflow réplique cela via des **contexts hiérarchiques avec héritage** :

```
~/.workflow/contexts/
├── _global/                ← Universal (commits, sécurité, tests)
├── mobile/                 ← Hérite _global
│   ├── flutter/            ← Hérite mobile (BLoC, go_router, Dart)
│   └── react-native/       ← Hérite mobile (RN patterns)
├── web/
│   └── nextjs/             ← Hérite web (App Router, RSC)
└── backend/
    └── fastapi/
```

À l'install, **seul `_global` existe**. Les autres contexts sont créés à la demande depuis des **templates bundled** quand `workflow init` détecte un nouveau type de projet :

```bash
$ cd ~/projects/my-flutter-app
$ workflow init "FoodDelivery"
🔍 Détecté : pubspec.yaml
   → contexte suggéré : mobile.flutter
   [O] Confirmer  [C] Choisir autre
$ O
✓ Active contexts : [_global, mobile, mobile.flutter]
```

**Multi-contexts actifs** : un projet active plusieurs contexts simultanément. Skills/decisions sont chargés du plus spécifique au plus général. Skills `USER_OVERRIDE` battent toujours les autres au scoring.

**Promotion cross-context** : le Curator détecte qu'un pattern présent dans `mobile.flutter` ET `mobile.react-native` est mobile-général → propose la promotion vers `mobile`.

**Portables** : `workflow context export mobile.flutter > my-flutter.tar.gz`. Un senior partage ses 80 skills accumulés à un junior.

---

## Les 6 Phases du Workflow Projet (revisitables, pas waterfall)

> **Principe fondamental** : ces phases ne sont pas un tunnel à sens unique. L'utilisateur peut **revenir** à une phase antérieure à tout moment (ajouter une fonctionnalité après avoir codé, changer la stack après une expérience), et **sauter** des phases via le **Starter Mode**.

### Starter Mode — coder en 30 secondes

Pour ne pas reproduire l'anti-pattern "1h de Q/R avant de pouvoir coder", Workflow propose un mode rapide :

```
workflow start --quick "ajoute un endpoint /users qui liste mes utilisateurs"
  → Workflow détecte qu'aucune phase n'a été faite
  → Génère une vision minimale + tech-stack par défaut détecté du projet
  → Crée TASK-001 directement, en parallèle commence Discovery en background
  → L'utilisateur code immédiatement, Workflow remplit les phases pendant ce temps
```

Les phases formelles (Discovery → Execution) restent disponibles via `workflow start --full` quand l'utilisateur veut une démarche structurée dès le départ (greenfield, projet d'équipe).

### Phase 1 — Discovery
L'utilisateur décrit son idée. Workflow pose des questions ciblées pour éliminer les zones d'ombre.
**Sortie** : `vision.md`
**Revisitable** : ajouter une zone manquante, pivoter le produit.

### Phase 2 — Specification
Workflow propose des fonctionnalités basées sur la vision, l'utilisateur valide ou ajuste.
**Sortie** : `features.json`
**Revisitable** : ajouter une fonctionnalité en cours de v1.0 (la nouvelle entrée crée une tâche pending au lieu de bloquer).

### Phase 3 — Design System
`DiscoveryPhase` et `DesignSystemPhase` capturent le style visuel souhaité. `design.json` est généré et utilisé par `ValidationPhase` pour produire des mockups ASCII conformes dans chaque tâche.
**Sortie** : `design.json` + `design-system.json` + `screen-flow.md`

### Phase 4 — Architecture
Workflow suggère la stack technologique avec justification (si non définie). Impose également :
- **TASK-001** : Setup projet + linter (systématique pour toute v1.0)
- **TASK-002** : Framework de tests + premier test de smoke (systématique pour toute v1.0)
- La policy `allowed_commands` initiale (whitelist + apprentissage — voir Pilier 6)

**Sortie** : `tech-stack.json`
**Revisitable** : changement de stack majeur (ajout de Redis, migration de Postgres → SQLite). Workflow détecte les conflits avec les décisions existantes via le decisions-graph.

### Phase 5 — Validation
Workflow génère les fichiers de tâches détaillés par version, l'utilisateur les valide ou demande des modifications.
**Sortie** : Dossier `versions/` avec les tâches validées.

**Règle de granularité sémantique** : `1 tâche = 1 PR mergeable atomiquement avec ses tests`. Le nombre exact de fichiers ou la durée estimée sont des indicateurs, pas des règles strictes — le critère réel est la *cohérence atomique* du diff produit.

### Phase 6 — Réalisation
Workflow exécute version par version, tâche par tâche, avec **CodePatcher chirurgical** (jamais de régénération de fichier complet) :

```
Consulter decisions.log scoré + skills pertinents
  → Génère un patch (search/replace blocks ou tool-based edit)
  → Applique → build_validate → Tests →
  ✅ Tout passe  → tâche done, decisions extraites, prochain
  ❌ Erreur      → SkillManager.find_fix_for_error() → retry avec known_fix (max 3)
  ❌ Retry réussi → ExecutionLoop crée un skill, Curator le consolidera plus tard
  ❌ Persistant  → escalade à l'user avec contexte complet + proposition de découpe
```

---

## Le `decisions.log` + `decisions-graph.json` — Journal Actif (Pilier 4)

Le `decisions.log` n'est pas un journal passif. Workflow le **consulte activement** avant de coder toute tâche impliquant des choix déjà faits (ORM, auth, base de données, framework, patterns).

```
# decisions.log — exemple

[2026-03-12] [TASK-004] ORM : Prisma choisi plutôt que TypeORM
  Raison : meilleure DX, migrations plus fiables sur PostgreSQL

[2026-03-18] [TASK-008] Authentification : JWT uniquement
  Raison : architecture stateless requise pour déploiement serverless
```

Workflow consulte avant de coder :
```
Workflow : "J'allais utiliser TypeORM, mais le decisions.log indique Prisma
           choisi le 12 mars pour des raisons de fiabilité des migrations.
           Je code avec Prisma."
```

### Graphe rétro-propageant — `decisions-graph.json`

Au-delà du log linéaire, Workflow maintient un **graphe** avec relations typées :

| Relation | Sens |
|---|---|
| `DEPENDS_ON` | A nécessite que B reste vrai |
| `CONTRADICTS` | A et B sont incompatibles (détecté automatiquement par LLM `role='fast'`) |
| `SUPERSEDES` | A remplace B (B obsolète) |
| `REFINES` | A précise/contraint B |

**Quand le dev change une décision**, Workflow détecte automatiquement les tâches `pending` impactées et **annote leur Journal** :

```
$ workflow decision update DEC-001 "Migrer Prisma → Drizzle"
↻ Rétro-propagation : 3 tâches impactées
  - TASK-018 : annoté "DEC-001 mis à jour, ré-évaluer la couche d'accès DB"
  - TASK-022 : annoté
  - TASK-027 : annoté
```

Niveaux de confiance : `USER_OVERRIDE` > `HIGH` > `MEDIUM` > `LOW`. Une décision `USER_OVERRIDE` bat toujours une décision LLM-inférée contradictoire.

---

## Format des Tâches Auto-Suffisantes

Chaque `TASK-XXX.md` contient tout ce dont Workflow a besoin pour l'exécuter **sans historique de conversation** :

```markdown
# TASK-003 : Authentification — JWT
## Version : v1.0

## Contexte Projet
Application: TaskFlow (API REST)
Stack: Node.js + Express + PostgreSQL + JWT
Phase: Backend — Sécurité

## User Story
EN TANT QU'utilisateur
JE VEUX pouvoir créer un compte et me connecter
AFIN D'accéder à mes tâches de manière sécurisée

## Intent
L'utilisateur veut que les clients puissent s'inscrire seuls, SANS passer par un admin.
L'email de confirmation est secondaire — ne pas bloquer l'inscription dessus.

## Préconditions
- filesExist: ["src/db/connection.js"]
- tasksCompleted: ["TASK-001", "TASK-002"]
- branch: "workflow/v1.0"
- testsPass: true

## Active Contexts
- _global, backend, backend.express

## Dépendances
- TASK-001 ✅ (setup projet + linter)
- TASK-002 ✅ (framework tests configuré)

## Fichiers à créer / modifier
- src/routes/auth.routes.js   [CRÉER]
- src/controllers/auth.js     [CRÉER]
- src/middleware/auth.js      [CRÉER]
- src/services/jwt.service.js [CRÉER]

## Critères d'acceptation
- [ ] POST /auth/register crée un user (email unique, mdp hashé bcrypt)
- [ ] POST /auth/login retourne un JWT valide 24h
- [ ] Middleware protège les routes avec Bearer token
- [ ] Tests unitaires sur les deux endpoints

## Mockup UI
(aucune interface — tâche backend / configuration)

## Journal
(vide — tâche jamais tentée)

## Statut
⬜ EN ATTENTE
```

Exemple de tâche avec interface (style minimaliste) :

```markdown
# TASK-007 : Page de connexion — UI
## Version : v1.0

...

## Mockup UI

### Écran — Login
┌─────────────────────────────────────┐
│                                     │
│            ◆ TaskFlow               │
│                                     │
│   Email                             │
│   ┌─────────────────────────────┐   │
│   │ john@example.com            │   │
│   └─────────────────────────────┘   │
│                                     │
│   Mot de passe                      │
│   ┌─────────────────────────────┐   │
│   │ ••••••••                    │   │
│   └─────────────────────────────┘   │
│                                     │
│   ┌─────────────────────────────┐   │
│   │       Se connecter          │   │
│   └─────────────────────────────┘   │
│                                     │
│   Pas encore de compte ? S'inscrire │
└─────────────────────────────────────┘
Style : Minimaliste — fond blanc #FFFFFF, texte #111827, bouton primaire #3B82F6, police Inter

...
```

> **Champ `Journal`** : Rempli automatiquement par Workflow à chaque report ou tentative partielle. Permet de savoir jusqu'où on est allé si la tâche a été interrompue.
> **Champ `Intent`** : Capture le "pourquoi humain". Injecté dans les prompts LLM avant les critères. Ne peut pas être vide si des critères d'acceptation sont définis.
> **Champ `Préconditions`** : Vérifié par `SyncChecker.checkPreconditions()` avant de démarrer la tâche. Généré automatiquement par `ValidationPhase`.
> **Champ `Mockup UI`** : Présent dans toutes les tâches. Pour les tâches avec interface, contient un ou plusieurs écrans en ASCII art respectant le style défini dans `design.json`. Pour les tâches backend/configuration, affiche `(aucune interface — tâche backend / configuration)`. Généré automatiquement par `ValidationPhase`.

---

## Reprise après Context Overflow

```
Nouvelle session — nouveau contexte LLM

SyncChecker :
  0. Valide la conformité au protocole .workflow/ (Pilier 1) — JSON Schema
  1. Vérifie que branche Git = version active dans .workflow/
  2. Compare état du repo avec progress.json
  3. Détecte fichiers modifiés manuellement
  4. Soumet le diff au LLM pour analyse sémantique
  5. Alerte si contradictions actives dans decisions-graph

Workflow : "Je reprends FreelanceApp — v1.5 ACTIVE (branche workflow/v1.5).
           Active contexts : [_global, web, web.nextjs].
           Tu as modifié src/auth/login.py depuis ma dernière session.
           Il semblerait que tu as codé la route /register.
           Je marque le critère 1 de TASK-003 comme validé.
           ⚠️ DEC-007 contredit DEC-002 (auth strategy) — résoudre ?
           Je reprends à partir de /login. C'est bien ça ?"

User : "Oui."
```

---

## Gestion des Versions

### Cycle de Vie
```
DRAFT → ACTIVE → COMPLETED → ARCHIVED
                     ↑
                HOTFIX (interrompt une version ACTIVE)
```

Une seule version **ACTIVE** à la fois. Workflow refuse de démarrer une v1.5 si la v1.0 n'est pas COMPLETED — sauf demande explicite ou hotfix.

### Couplage Git

```
workflow version create v1.5 "Export PDF"
  → git checkout -b workflow/v1.5

workflow version switch v1.5
  → ❌ Repo non propre → message explicite → user doit commiter
  → ✅ Repo propre → git checkout workflow/v1.5 → rechargement contexte v1.5

workflow version complete
  → git merge workflow/v1.5 → main
  → Bilan automatique → prépare version suivante en DRAFT
```

**Règle absolue** : Workflow ne stashe jamais. Explicite > Magique.

### Hotfix

```
v1.5 ACTIVE, bug critique détecté sur v1.0 en prod.

workflow version hotfix v1.0.1 "Correction calcul TVA"
  → Workflow bloque si modifications non commitées sur v1.5
  → Crée workflow/hotfix/v1.0.1 depuis workflow/v1.0
  → Corrige, teste, livre
  → Propose d'intégrer le fix dans la v1.5
```

### Bilan Automatique
```
v1.0 COMPLETED

Workflow : "La v1.0 est terminée.
           ✅ 12 tâches complétées
           ⚠️  2 tâches reportées en v1.5 (TASK-011, TASK-012)

           Je démarre la v1.5 ? (git checkout workflow/v1.5)"
```

---

## Observation Passive et Conscience Continue

### `workflow watch` — Mode Annotation Passive

Quand l'utilisateur code manuellement, `workflow watch` observe les modifications via `watchfiles` et pose des questions de clarification sous forme de fichiers dans `.workflow/questions/`. Aucune interruption : les questions s'accumulent silencieusement et sont traitées au prochain `workflow start`.

### `DaemonHeartbeat` — Surveillance Continue

Un daemon tourne en arrière-plan et génère un briefing matin dans `.workflow/briefings/YYYY-MM-DD.md`. Il surveille :
- L'état des tâches en cours
- Les contradictions actives dans `decisions-graph`
- Les builds CI distants
- **Lance automatiquement le `SkillCurator`** quand ≥10 nouveaux skills depuis la dernière run

### `ParallelExecutor` — Exécution Parallèle (Phase 7)

Quand le graphe de dépendances révèle 3 tâches indépendantes, Workflow lance 3 instances `ExecutionLoop` simultanément via **git worktrees isolés**, puis merge les branches dans la branche version. Aucun outil concurrent ne fait ça aujourd'hui.

### `workflow onboard` — Onboarding Nouveau Développeur

Un nouveau développeur sur le projet peut comprendre l'état complet en 30 secondes via `workflow onboard`, qui lit tous les fichiers `.workflow/` (vision, features, tech-stack, décisions, skills, contexts actifs, prochaine tâche prête, contradictions à connaître) et génère un résumé structuré.

---

## Sécurité MCP — `AllowedCommandsPolicy` à 3 niveaux (Pilier 6)

Workflow valide chaque commande shell avant exécution selon une **politique en 3 niveaux** — pas une whitelist gelée :

### Niveau 1 — Auto-allow (built-in)
Commandes universellement sûres pré-approuvées (`git status`, `ls`, `pytest`, `npm test`, etc.). Exécutables sans demander.

### Niveau 2 — Project allowed (`.workflow/allowed-commands.json`)
Commandes approuvées au niveau du projet, partagées via git :
```json
{
  "allowed": [
    {"command": "uv run alembic", "scope": "prefix", "approved_by": "alister", "approved_at": "..."},
    {"command": "docker compose up", "scope": "exact"}
  ],
  "denied": [
    {"command": "rm -rf", "reason": "trop dangereux"}
  ]
}
```

### Niveau 3 — Prompt à l'utilisateur (apprentissage)
Pour toute commande hors Niveau 1/2 :
```
Workflow : "Je veux exécuter : prisma migrate dev --name add-users"
           Approuver ?
           [O] Oui, une fois
           [P] Oui et mémoriser (préfixe : 'prisma migrate')
           [E] Oui et mémoriser (exact)
           [N] Non
           [B] Non et bannir
```

En CI/non-interactif, les commandes inconnues sont **DENY par défaut** (pas de hang).

**ALWAYS_DENIED hardcodé** : `rm -rf /`, `sudo rm`, `chmod 777`, `curl | sh`, `dd if=`. Non-overridable.

Exemples multi-stack :
- Python → `uv run ruff check . && uv run mypy src/`
- Node.js → `npm run lint && npm run build`
- Rust → `cargo clippy && cargo test`
- .NET → `dotnet build && dotnet test`

---

## Structure `.workflow/` Complète

```
.workflow/
├── project.json               # active_contexts: [_global, mobile, mobile.flutter] (Pilier 7)
├── vision.md
├── features.json
├── tech-stack.json
├── design.json                # Style, couleurs, références, mockups
├── code-index.json            # Index symboles — Phase 9
├── decisions.log              # Journal texte brut — lisible humain
├── decisions-graph.json       # Relations entre décisions (CONTRADICTS, DEPENDS_ON, SUPERSEDES, REFINES)
├── allowed-commands.json      # Whitelist + apprentissage (Pilier 6)
├── skills/                    # Skills strictement projet (le reste vit dans les contexts globaux)
│   └── *.md
├── questions/                 # WatchMode — questions en attente de réponse
│   └── YYYY-MM-DD-Q001.md
├── briefings/                 # DaemonHeartbeat — briefings quotidiens
│   └── YYYY-MM-DD.md
└── versions/
    └── v1.0/
        ├── meta.json
        ├── progress.json
        └── tasks/
            └── TASK-001.md
```

**Et au niveau global** (`~/.workflow/`) :
```
~/.workflow/
├── config.yaml                          # Routage LLM par défaut
├── ingestions.log                       # Audit trail des project ingestions
└── contexts/                            # Pilier 7 — expertises spécialisées
    ├── _global/                         # Universal patterns
    ├── mobile/                          # Hérite _global
    │   └── flutter/                     # Hérite mobile
    ├── web/
    │   └── nextjs/
    └── backend/
        └── fastapi/
```

---

## Commandes CLI Complètes

### Cycle projet
```bash
workflow init "Nom"                          # Initialise .workflow/ + détecte context
workflow start                               # Lance/reprend les phases (full mode)
workflow start --quick "description"         # Starter Mode (saut direct à ACTIVE)
workflow start --revisit DISCOVERY           # Revenir à une phase antérieure
workflow run                                 # Exécute la prochaine tâche prête
workflow status                              # État du projet (phase, version, contexts, done/pending)
workflow onboard                             # Onboarding nouveau dev en 30 secondes
```

### Apprentissage actif (Pilier 2 — 4 sources)
```bash
# Source 2 : workflow teach / avoid (user_explicit → USER_OVERRIDE)
workflow teach "go_router toujours, pas Navigator 1.0"
workflow teach --context mobile.flutter "j'utilise BLoC, pas Provider"
workflow teach --tags "auth,jwt" "rotation refresh tokens 30j"
workflow teach --personal "j'aime semicolons explicites en JS"
workflow avoid "stop class-based React components"
workflow avoid --reason "FastAPI a Depends" "pas de @middleware auth"

# Source 4 : workflow learn-from (project_ingestion)
workflow learn-from ~/projects/old-app --context web.nextjs \
  --learn architecture,naming,patterns --ignore styling,deps
workflow learn-from ~/projects/old-app --dry-run        # voir avant ingérer

# Curator (consolidation skills)
workflow curate                              # consolide skills (dedup, promote, archive)
workflow curate --dry-run                    # plan sans appliquer

# Source 3 : in_flow_correction (pendant que l'agent code)
# Pas de commande — Ctrl+T pendant `workflow run` ou `/correct` MCP
```

### Contexts (Pilier 7)
```bash
workflow context list                        # contexts actifs chez l'utilisateur
workflow context list --available            # templates bundled non encore créés
workflow context create mobile.flutter       # depuis template bundled
workflow context create game.bevy --parent _global   # context custom
workflow context activate mobile.react-native --temporary   # ajouter au projet courant
workflow context fork web my-custom-web      # cloner un context existant
workflow context promote mobile.flutter:auth-pattern → mobile   # promotion manuelle
workflow context export mobile.flutter > my-flutter.tar.gz
workflow context install ~/Downloads/some-context.tar.gz
workflow context delete game.bevy            # refuse _global
```

### Versions
```bash
workflow version list
workflow version create v1.5 "Description"
workflow version switch v1.5                 # bloque si repo non propre
workflow version complete                    # merge + bilan auto + prépare suivante
workflow version hotfix v1.0.1 "raison"     # branche hotfix depuis parent
```

### Décisions
```bash
workflow decisions search "auth"             # full-text SQLite FTS5
workflow decisions graph                     # affiche le graphe + contradictions
workflow decisions update DEC-001 "nouvelle décision"   # rétro-propage aux tâches pending
```

### Allowed commands (Pilier 6)
```bash
workflow allow "uv run alembic"              # ajouter à la whitelist projet (scope=prefix)
workflow allow --exact "docker compose up"   # match exact uniquement
workflow deny "rm -rf"                       # ajouter à la liste denied
workflow allow list                          # voir la politique courante
```

### Couches proactives (Phase 7)
```bash
workflow watch                               # mode annotation passive (watchfiles)
workflow daemon start                        # lance DaemonHeartbeat en arrière-plan
workflow daemon stop
workflow parallel run                        # exécute tâches indépendantes en parallèle (worktrees)
```

### Polish (Phase 10)
```bash
workflow doc generate                        # README + ARCHITECTURE.md + CHANGELOG auto
workflow audit                               # divergences code/tâches + contradictions
workflow estimate [version]                  # estimation basée sur l'historique git réel
```

### Surfaces tierces (Phase 8)
```bash
workflow mcp serve                           # démarre le serveur MCP (pour Claude Code/Cursor)
workflow rest serve                          # API REST locale (FastAPI)
workflow telegram start                      # bot Telegram (briefing + commandes)
```
