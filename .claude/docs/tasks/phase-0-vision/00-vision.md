# Phase 0 — Vision Produit

## Le Problème

Les agents de code actuels souffrent d'un défaut structurel : ils vivent dans le contexte de conversation. Quand ce contexte sature, l'agent perd le fil. L'utilisateur recommence tout depuis zéro dans une nouvelle session. Pire : ces agents ne comprennent jamais un projet dans sa globalité — on leur soumet des tâches isolées sans vision d'ensemble.

## La Solution : Workflow

**Workflow** est un agent de code conçu pour un freelance ou développeur indépendant. Sa proposition de valeur unique : **la mémoire projet persiste entre toutes les sessions**.

Toute la connaissance du projet (vision, fonctionnalités, stack, décisions, état d'avancement) vit dans un dossier `.workflow/` à la racine du projet. Au démarrage de chaque session, Workflow lit ces fichiers et reprend exactement où il s'était arrêté — sans que l'utilisateur ait besoin de réexpliquer quoi que ce soit.

---

## Deux Modes d'Existence

### Workflow Agent
Agent autonome complet. L'utilisateur interagit avec lui comme avec un développeur-réalisateur. Workflow est à la fois le LLM (via Claude API) et le gestionnaire de projet. Disponible en CLI ou via Telegram.

### Workflow Core
Workflow comme gestionnaire de projet pur, intégrable dans d'autres agents via MCP. Dans ce mode, Workflow n'est **pas** le LLM — il est la mémoire, la structure et les outils. Le modèle qui réfléchit, c'est l'agent hôte (Claude Code, Cursor, etc.).

> **Stratégie** : Construire Workflow Core en premier. C'est le chemin le plus court vers quelque chose d'utilisable — branché sur Claude Code en quelques semaines.

---

## Les 5 Phases du Workflow Projet

### Phase 1 — Discovery
L'utilisateur décrit son idée. Workflow pose des questions ciblées pour éliminer les zones d'ombre.
**Sortie** : `vision.md`

### Phase 2 — Specification
Workflow propose des fonctionnalités basées sur la vision, l'utilisateur valide ou ajuste.
**Sortie** : `features.json`

### Phase 3 — Validation
Workflow génère les fichiers de tâches détaillés par version, l'utilisateur les valide ou demande des modifications.
**Sortie** : Dossier `versions/` avec les tâches de chaque version validées.

**Règle de granularité** : `1 tâche = 1 PR potentielle = max 4h de travail = max 3 fichiers créés/modifiés`. Si une tâche dépasse ces seuils, Workflow la découpe automatiquement avant validation.

### Phase 4 — Architecture
Workflow suggère la stack technologique avec justification (si non définie). Impose également :
- **TASK-001** : Setup projet + linter (systématique pour toute v1.0)
- **TASK-002** : Framework de tests + premier test de smoke (systématique pour toute v1.0)
- La liste `allowed_commands` dans `tech-stack.json`

**Sortie** : `tech-stack.json`

### Phase 5 — Réalisation
Workflow exécute version par version, tâche par tâche.

```
Consulter decisions.log
  → Code → build_validate → Tests →
  ✅ Tout passe  → tâche done, tâche suivante
  ❌ Erreur      → Workflow analyse → consulte decisions.log → corrige → retente (max 3)
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

## Journal
(vide — tâche jamais tentée)

## Statut
⬜ EN ATTENTE
```

> **Champ `Journal`** : Rempli automatiquement par Workflow à chaque report ou tentative partielle. Permet de savoir jusqu'où on est allé si la tâche a été interrompue.

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
