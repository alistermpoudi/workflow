# Phase 0 — Vision Produit

## Le Problème

Les agents de code actuels souffrent d'un défaut structurel : ils vivent dans le contexte de conversation. Quand ce contexte sature, l'agent perd le fil. L'utilisateur recommence tout depuis zéro dans une nouvelle session. Pire : ces agents ne comprennent jamais un projet dans sa globalité — on leur soumet des tâches isolées sans vision d'ensemble. Et même quand un agent résout brillamment un problème, **il oublie tout au projet suivant** : aucune capitalisation d'expérience.

## La Solution : Workflow

**Workflow** est un agent de code pour freelance et développeur indépendant qui **se souvient et accumule de l'expérience**, comme un humain qui a fait 30 projets et applique ses leçons.

Deux mémoires distinctes, persistantes :

1. **Mémoire projet** (`.workflow/` dans le projet) — vision, fonctionnalités, stack, décisions, état d'avancement. Workflow reprend exactement où il s'est arrêté à chaque session.
2. **Mémoire institutionnelle** (`~/.workflow/skills/` global) — patterns appris à travers tous les projets. Chaque retry réussi devient un skill ; un Curator LLM consolide périodiquement. **Workflow devient plus intelligent à chaque projet.**

> **Thèse différenciatrice** : Cursor / Claude Code sont des "assistants experts dans la session". Workflow est un **PM technique avec mémoire institutionnelle**. Quand tu démarres un projet React, il sait déjà comment tu structures tes hooks, quelles erreurs Prisma reviennent toujours, et quels choix d'architecture tu as faits sur les 5 derniers projets similaires.

### Les 6 Piliers Load-Bearing

1. **Protocole `.workflow/`** — schéma versionné lisible par tous les clients (CLI, VS Code, Telegram, web). Pas un dossier ad-hoc.
2. **Skills + Curator** — boucle d'apprentissage cumulé cross-projet. Le vrai différenciateur.
3. **Multi-LLM par rôle** — `reasoning` pour penser, `code_generation` pour coder, `fast` pour scorer, `curator` pour consolider. Chaque rôle son optimum.
4. **Décisions actives + graphe rétro-propageant** — `decisions.log` + graphe de relations (CONTRADICTS, DEPENDS_ON, SUPERSEDES). Détecte les contradictions et propage aux tâches en cours.
5. **CodePatcher chirurgical** — diffs et search/replace blocks dès le départ. Jamais de régénération de fichier entier.
6. **MCP Server comme surface primaire** — toutes les autres surfaces (CLI, VS Code, Telegram, REST) sont des clients du protocole MCP.

> Détail complet : voir `02-pillars.md`.

---

## Deux Modes d'Existence

### Workflow Agent
Agent autonome complet. L'utilisateur interagit avec lui comme avec un développeur-réalisateur. Workflow est à la fois le LLM (via Claude API) et le gestionnaire de projet. Disponible en CLI ou via Telegram.

### Workflow Core
Workflow comme gestionnaire de projet pur, intégrable dans d'autres agents via MCP. Dans ce mode, Workflow n'est **pas** le LLM — il est la mémoire, la structure et les outils. Le modèle qui réfléchit, c'est l'agent hôte (Claude Code, Cursor, etc.).

> **Stratégie** : Construire Workflow Core en premier. C'est le chemin le plus court vers quelque chose d'utilisable — branché sur Claude Code en quelques semaines.

---

## Les 5 Phases du Workflow Projet (revisitables, pas waterfall)

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

## Le `decisions.log` — Journal Actif

Le `decisions.log` n'est pas un journal passif. Workflow le **consulte activement** avant de coder toute tâche impliquant des choix déjà faits (ORM, auth, base de données, framework, patterns).

```
# decisions.log — exemple

[2025-03-12] [TASK-004] ORM : Prisma choisi plutôt que TypeORM
  Raison : meilleure DX, migrations plus fiables sur PostgreSQL

[2025-03-18] [TASK-008] Authentification : JWT uniquement
  Raison : architecture stateless requise pour déploiement serverless
```

Workflow consulte avant de coder :
```
Workflow : "J'allais utiliser TypeORM, mais le decisions.log indique Prisma
           choisi le 12 mars pour des raisons de fiabilité des migrations.
           Je code avec Prisma."
```

Chaque décision est enregistrée avec : date, tâche d'origine, décision, raison.

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
- filesExist: ["src/tools/FileSystem.js"]
- tasksCompleted: ["TASK-001", "TASK-002"]
- branch: "workflow/v1.0"
- testsPass: true

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
  1. Vérifie que branche Git = version active dans .workflow/
  2. Compare état du repo avec progress.json
  3. Détecte fichiers modifiés manuellement
  4. Soumet le diff au LLM pour analyse sémantique

Workflow : "Je reprends FreelanceApp — v1.5 ACTIVE (branche workflow/v1.5).
           Tu as modifié src/controllers/auth.js depuis ma dernière session.
           Il semblerait que tu as codé la route /register.
           Je marque le critère 1 de TASK-003 comme validé.
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

Quand l'utilisateur code manuellement, `workflow watch` observe les modifications via `chokidar` et pose des questions de clarification sous forme de fichiers dans `.workflow/questions/`. Aucune interruption : les questions s'accumulent silencieusement et sont traitées au prochain `workflow start`.

### `DaemonHeartbeat` — Surveillance Continue

Un daemon tourne en arrière-plan et génère un briefing matin dans `.workflow/briefings/YYYY-MM-DD.md`. Il surveille l'état du projet et alerte sur les dérives.

### `workflow onboard` — Onboarding Nouveau Développeur

Un nouveau développeur sur le projet peut comprendre l'état complet en 30 secondes via `workflow onboard`, qui lit tous les fichiers `.workflow/` et génère un résumé structuré.

---

## Sécurité MCP — `allowed_commands`

Workflow n'exécute via MCP que les commandes explicitement listées dans `tech-stack.json#allowed_commands`. Toute autre commande est rejetée, loggée, et requiert une confirmation manuelle de l'utilisateur.

```json
{
  "build_validate": "npm run lint && npm run build",
  "test": "npm test",
  "allowed_commands": [
    "npm run lint",
    "npm run build",
    "npm test",
    "npx prisma migrate"
  ]
}
```

Exemples multi-stack :
- Node.js → `npm run lint && npm run build`
- Python → `pylint src/ && python -m pytest`
- Rust → `cargo clippy && cargo test`
- .NET → `dotnet build && dotnet test`

---

## Structure `.workflow/` Complète

```
.workflow/
├── project.json
├── vision.md
├── features.json
├── tech-stack.json
├── design.json                # Préférences visuelles — style, couleurs, références, mockups
├── code-index.json
├── decisions.log              # Journal texte brut — lisible humain
├── decisions-graph.json       # Relations entre décisions (CONTRADICTS, DEPENDS_ON, SUPERSEDES, REFINES)
├── failure-patterns.json      # Erreurs connues + solutions cross-tâches
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

---

## Commandes CLI Complètes

```bash
workflow init "Nom"              # Initialise un projet
workflow start                   # Lance/reprend les phases
workflow run                     # Exécute la prochaine tâche
workflow status                  # Affiche l'état du projet
workflow watch                   # Mode annotation passive (WatchMode + chokidar)
workflow daemon                  # Lance le DaemonHeartbeat en arrière-plan
workflow onboard                 # Onboarding nouveau dev en 30 secondes
workflow doc generate            # Génère README + ARCHITECTURE.md + CHANGELOG
workflow audit                   # Détecte les divergences code/tâches
workflow estimate [version]      # Estimations basées sur l'historique git réel
workflow version list
workflow version create v1.5 "Description"
workflow version switch v1.5
workflow version complete
workflow version hotfix v1.0.1 "raison"
```
