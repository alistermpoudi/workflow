# 🤖 CLAUDE.md — Guide pour Claude Code

## À propos de ce fichier

Ce fichier contient toutes les instructions et le contexte nécessaires pour travailler efficacement sur **Workflow**. Lis ce fichier en premier avant toute intervention.

---

## 🎯 Qu'est-ce que Workflow ?

**Workflow** est un agent de code conçu pour les développeurs freelance et toute personne souhaitant créer une application mobile, desktop ou web. Sa particularité fondamentale : il ne dépend **jamais du contexte de conversation** pour se souvenir du projet.

### Le Problème à Résoudre

Les agents de code actuels (Claude Code, Cursor, Codex, etc.) souffrent d'un défaut structurel : ils vivent dans le contexte de conversation. Quand ce contexte sature, l'agent perd le fil du projet. L'utilisateur doit tout réexpliquer depuis zéro.

**Workflow résout exactement ce problème.**

### La Solution

Toute la connaissance projet est persistée dans des fichiers structurés dans un dossier `.workflow/` à la racine du projet cible. Au démarrage de chaque session, Workflow lit ces fichiers et sait exactement où il en est — sans aucune réexplication.

### Les 5 Piliers de l'Indispensabilité

1. **Daemon Heartbeat** — tourne en arrière-plan, envoie un briefing matin, surveille les builds, propose la prochaine tâche automatiquement
2. **Présence partout** — CLI, Telegram, VS Code extension (sidebar + annotations inline), GitHub integration
3. **Cerveau d'équipe** — `.workflow/` partagé via git, `workflow onboard` pour onboarding instantané des nouveaux devs, détection de conflits de décisions entre devs
4. **Source de vérité unique** — `workflow doc generate` (README + ARCHITECTURE.md + CHANGELOG auto), `workflow audit` (divergence code/tâches), `workflow estimate` (basé sur temps réel par tâche)
5. **Écosystème** — WorkflowLibrary cross-projet, workflow-hub (marketplace de patterns communautaire)

---

## 🏗️ Deux Modes d'Existence

| Mode | Description | Qui réfléchit |
|------|-------------|---------------|
| **Workflow Agent** | Agent autonome complet — CLI interactive ou Telegram | Workflow (Claude API) |
| **Workflow Core** | Gestionnaire de projet pur exposé via MCP | L'agent hôte (Claude Code, Cursor, etc.) |

Dans les deux cas : même cerveau, mêmes fichiers `.workflow/`, mêmes outils. Seule la couche d'interaction change.

> ⚠️ **Stratégie de build** : Commencer par **Workflow Core (MCP)** — c'est le chemin le plus court vers quelque chose d'utilisable. Workflow Core branché sur Claude Code donne immédiatement de la valeur, avant même que le Workflow Agent autonome existe.

---

## 📁 Structure du Projet

```
workflow/
├── CLAUDE.md               # CE FICHIER
├── package.json
├── .gitignore
│
├── src/
│   ├── core/               # Cerveau — indépendant de l'interface
│   │   ├── WorkflowAgent.js        # Orchestrateur principal (Agent mode)
│   │   ├── ProjectMemory.js        # Lecture/écriture .workflow/
│   │   ├── VersionManager.js       # Cycle de vie versions + pilote GitManager
│   │   ├── DecisionsLog.js         # Journal actif — consulté avant chaque tâche
│   │   ├── ContextManager.js       # Hiérarchie de chargement contexte LLM
│   │   ├── PhaseManager.js         # Orchestre les 5 phases
│   │   ├── SyncChecker.js          # State drift + vérif branche Git
│   │   ├── DaemonHeartbeat.js      # Daemon proactif (briefing + surveillance)
│   │   └── WatchMode.js            # Annotation passive (questions sans interruption)
│   │
│   ├── phases/             # Les 5 phases projet
│   │   ├── DiscoveryPhase.js       # Phase 1 : questions ciblées → vision.md
│   │   ├── SpecificationPhase.js   # Phase 2 : fonctionnalités → features.json
│   │   ├── ValidationPhase.js      # Phase 3 : tâches détaillées → versions/
│   │   ├── ArchitecturePhase.js    # Phase 4 : stack + TASK-001/002 + allowed_commands
│   │   └── ExecutionPhase.js       # Phase 5 : code → validate → corriger → done
│   │
│   ├── tools/              # Outils techniques
│   │   ├── CodePatcher.js          # Diffs chirurgicaux + fallback AST (v1.5)
│   │   ├── CodeIndexer.js          # Index JSON + variantes LLM + ripgrep (v1.5)
│   │   ├── ExecutionLoop.js        # build_validate générique → test → correction
│   │   ├── FileSystem.js           # Opérations fichiers
│   │   ├── GitManager.js           # Piloté par VersionManager
│   │   └── TaskManager.js          # CRUD sur les fichiers de tâches
│   │
│   ├── interfaces/         # Couches d'interaction
│   │   ├── CLI.js                  # Console interactive (readline + chalk)
│   │   ├── TelegramBot.js          # Bot Telegram (v2)
│   │   ├── MCPServer.js            # Workflow Core — validation allowed_commands
│   │   ├── RestAPI.js              # API REST locale (v2)
│   │   ├── PipeCLI.js              # CLI pipe (v2)
│   │   ├── VSCodeExtension/        # Extension VS Code (sidebar + annotations inline)
│   │   └── GitHubIntegration.js    # Webhooks GitHub/GitLab
│   │
│   └── llm/                # Abstraction LLM
│       ├── LLMProvider.js          # Multi-LLM (Claude par défaut)
│       └── PromptBuilder.js        # Prompts avec contexte projet injecté
│
└── .claude/
    ├── PROGRESS.md         # Suivi de progression — lire en début de session
    └── docs/
        └── tasks/          # Tâches détaillées par phase de build
```

---

## 📐 Structure `.workflow/` (ce que Workflow génère dans les projets cibles)

```
.workflow/
├── project.json            # Métadonnées projet
├── vision.md               # Vision produit (sortie Phase 1)
├── features.json           # Fonctionnalités validées (sortie Phase 2)
├── tech-stack.json         # Stack + build_validate + test + allowed_commands
├── design.json             # Style visuel — style, couleurs, références, notes (sortie DiscoveryPhase)
├── code-index.json         # Index fonctions/classes/exports (mis à jour en continu)
├── decisions.log           # Journal actif des décisions techniques
└── versions/
    ├── v1.0/
    │   ├── meta.json       # titre, description, statut, branche git
    │   ├── tasks/
    │   │   ├── TASK-001.md
    │   │   ├── TASK-002.md
    │   │   └── ...
    │   └── progress.json
    ├── v1.5/
    └── v2.0/
```

---

## 🛠️ Stack Technique

| Composant | Choix | Raison |
|-----------|-------|--------|
| Runtime | Node.js | Écosystème riche, async natif |
| LLM | Anthropic Claude API (claude-sonnet-4-6) | Meilleur pour le code, large context |
| CLI MVP | `readline` + `chalk` | Simple, zéro overhead, débogable facilement |
| CLI v1.5 | Ink (React terminal) | UI console riche après validation MVP |
| Telegram | `node-telegram-bot-api` | Simple et stable (v2) |
| MCP | `@modelcontextprotocol/sdk` | Standard officiel Anthropic |
| Analyse AST | `tree-sitter` | Parsing multi-langage robuste (v1.5) |
| Recherche code | `ripgrep` (subprocess) | Plus fiable qu'un regex custom sur l'index JSON |
| Fichiers projet | Markdown + JSON | Lisibles par humain ET machine |
| Config | `workflow.config.js` (JS/YAML) | Flexible |
| Tests | Vitest | Rapide, ESM natif |
| Watcher fichiers | `chokidar` | WatchMode — surveillance modifications |
| VS Code | `@vscode/extension-api` | Extension sidebar + annotations inline |
| GitHub | `@octokit/rest` | Webhooks + sync PR/issues |

---

## 📋 Les 8 Phases de Build

| Phase | Nom | Scope | Roadmap cible |
|-------|-----|-------|---------------|
| 0 | Vision & Architecture | Documentation | ✅ Complété |
| 1 | Foundation | `ProjectMemory`, `DecisionsLog`, `SyncChecker`, `TaskManager`, `FileSystem`, `GitManager` | MVP |
| 2 | Les 5 Phases Projet | `DiscoveryPhase` → `ExecutionPhase` + `PhaseManager` | MVP |
| 3 | Exécution de Base | `ExecutionLoop`, CLI readline, `DaemonHeartbeat`, `WatchMode` | MVP |
| 4 | MCP Server (Workflow Core) | `MCPServer.js` — tous les outils MCP exposés | MVP |
| 5 | Présence & Intégrations | VS Code Extension, GitHub Integration, Team Onboarding | V1 |
| 6 | Robustesse | `CodePatcher`, `CodeIndexer`, `ContextManager` avancé, `WorkflowLibrary` | V1.5 |
| 7 | Génération & Audit | `workflow doc generate`, `workflow audit`, `workflow estimate` | V2 |
| 8 | Écosystème | Telegram, REST API, CLI Ink, workflow-hub (marketplace) | V3 |

---

## 🔑 Concepts Clés Non-Négociables

### 1. La Hiérarchie de Chargement de Contexte

À chaque démarrage/reprise, `ContextManager` charge dans cet ordre strict :

```
Système (toujours — ~500 tokens max) :
  project.json + vision.md résumé + tech-stack.json

Version active (au switch) :
  meta.json + progress.json (tâches done/pending, sans le contenu)

Tâche courante (au start task) :
  TASK-XXX.md complet
  Fichiers listés dans "Fichiers à modifier" (lecture sélective)
  5 dernières entrées du decisions.log

On-demand :
  CodeIndexer.search() pour fonctions pertinentes
  Erreur de build si en boucle de correction
```

**Ne jamais charger l'intégralité du projet en contexte** — c'est exactement le problème qu'on résout.

### 2. Le `decisions.log` est Actif, pas Passif

Workflow **consulte** le log avant de coder une tâche impliquant des choix déjà faits (ORM, auth, DB, framework). Il ne fait pas que l'écrire.

```
Workflow : "J'allais utiliser TypeORM, mais le decisions.log indique Prisma
           choisi le 12 mars. Je code avec Prisma."
```

### 3. Règle de Granularité des Tâches

```
1 tâche Workflow = 1 PR potentielle = max 4h de travail = max 3 fichiers créés/modifiés
```

Si une tâche dépasse ces seuils, `ValidationPhase` la découpe automatiquement avant validation par l'utilisateur.

### 4. `allowed_commands` — Sécurité MCP

`MCPServer.js` **ne peut exécuter que les commandes listées** dans `tech-stack.json#allowed_commands`. Toute autre commande est rejetée et loggée. Aucune exception.

### 5. Versioning Git — Règle "No Stash"

Workflow **ne stashe jamais automatiquement**. Si le répertoire n'est pas propre lors d'un `version switch`, Workflow bloque avec un message explicite et demande à l'utilisateur de commiter.

### 6. Le Format des Tâches Auto-Suffisantes

Chaque `TASK-XXX.md` doit pouvoir être exécuté **sans historique de conversation** :

```markdown
# TASK-XXX : Titre
## Version : vX.Y

## Contexte Projet
Application, stack, phase courante

## User Story
EN TANT QUE / JE VEUX / AFIN DE

## Intent
Pourquoi l'utilisateur veut vraiment cette fonctionnalité — guide les décisions d'implémentation ambiguës.

## Préconditions
- filesExist: [...]
- tasksCompleted: [...]
- branch: "workflow/vX.Y"

## Dépendances
- TASK-001 ✅
- TASK-002 ✅

## Fichiers à créer / modifier
- src/foo.js [CRÉER]
- src/bar.js [MODIFIER]

## Critères d'acceptation
- [ ] Critère 1
- [ ] Critère 2

## Mockup UI

### Écran — [Nom de l'écran]
┌──────────────────────────────┐
│  [contenu de l'écran]        │
└──────────────────────────────┘
Style : [style choisi] — [palette, typographie]

[ou pour les tâches backend :]
(aucune interface — tâche backend / configuration)

## Journal
[date] Reportée depuis vX.X — raison : ...
[date] Tentative partielle : fichier src/foo.js créé, manque ...

## Statut
⬜ EN ATTENTE | 🔄 EN COURS | ✅ TERMINÉ | ❌ REPORTÉ
```

> **Le champ `Journal`** est rempli automatiquement par Workflow à chaque report ou tentative partielle.
> **Le champ `Mockup UI`** est généré automatiquement par `ValidationPhase` en respectant le style de `design.json`. Toujours présent — `(aucune interface)` pour les tâches sans UI.

### 7. TASK-001 et TASK-002 sont Systématiques

Les deux premières tâches de toute v1.0 sont imposées par `ArchitecturePhase` :
- **TASK-001** : Setup du projet et configuration du linter
- **TASK-002** : Configuration du framework de tests avec un premier test de smoke

### 8. Reprise après Context Overflow

```
SyncChecker au démarrage :
1. Vérifie que la branche Git = version active dans .workflow/
2. Compare état du repo avec progress.json
3. Détecte fichiers modifiés manuellement → soumet diff au LLM
4. Annonce l'état exact et demande confirmation avant de reprendre
```

### 9. Le Daemon est Toujours Présent

`DaemonHeartbeat` tourne en arrière-plan (launchd/systemd). Il envoie un briefing au démarrage de la journée, surveille les builds CI, et propose la prochaine tâche quand une est terminée. Il ne bloque jamais l'utilisateur.

### 10. Watch Mode — Annotation Sans Interruption

`WatchMode` observe les fichiers du projet. Quand un fichier est créé/modifié hors du scope d'une tâche, il crée un fichier question dans `.workflow/questions/`. L'utilisateur répond en éditant le fichier (oui/non/plus-tard). Zéro interruption.

### 11. `workflow onboard` — Onboarding Instantané

Un nouveau développeur sur le projet lance `workflow onboard`. Workflow lit tout le `.workflow/` et génère : résumé du projet, stack expliquée avec raisons, état d'avancement, les 5 décisions clés à connaître, première tâche suggérée. Objectif : autonomie en 30 secondes.

### 12. Design Style — Capturé Dès la Discovery

`DiscoveryPhase` demande obligatoirement le style de design souhaité (minimaliste, material, glassmorphism, néomorphisme, brutaliste, etc.) avant de terminer. Le choix est persisté dans `design.json`. `ValidationPhase` l'utilise pour générer des mockups ASCII dans chaque tâche impliquant une interface — **le développeur sait à quoi doit ressembler chaque écran avant même de coder**.

Styles disponibles : Minimaliste · Material Design · Glassmorphism · Néomorphisme · Brutaliste · Doux/Pastel · Dashboard Pro · Mobile-First · Cyberpunk · Personnalisé.

```json
// design.json — exemple
{
  "style": "minimaliste",
  "styleLabel": "Minimaliste",
  "colorScheme": "clair",
  "references": ["Linear", "Stripe"],
  "customNotes": null,
  "collectedAt": "2026-04-05T10:23:00Z"
}
```

---

## 🔗 Dépendances Entre Phases de Build

```
Phase 0 (Vision & Architecture)
         │
         ▼
Phase 1 (Foundation — ProjectMemory, DecisionsLog, SyncChecker, TaskManager)
         │
         ▼
Phase 2 (Les 5 Phases Projet — Discovery → Execution)
         │
         ▼
Phase 3 (Exécution de Base — ExecutionLoop, CLI readline, DaemonHeartbeat, WatchMode)
         │
         ▼
Phase 4 (MCP Server — Workflow Core complet)
         │
         ▼
Phase 5 (Présence & Intégrations — VS Code, GitHub, Onboarding)
         │
         ▼
Phase 6 (Robustesse — CodePatcher, CodeIndexer, WorkflowLibrary)
         │
    ┌────┴────┐
    ▼         ▼
Phase 7    Phase 8
(Génération & Audit)  (Écosystème)
```

---

## ⚠️ Règles NON-NÉGOCIABLES

1. **`allowed_commands` est une whitelist stricte** — jamais de commande shell dynamique sans être dans la liste
2. **Jamais de stash automatique** — bloquer et expliquer si répertoire non propre
3. **Toujours consulter `decisions.log`** avant de coder une tâche impliquant des choix d'architecture
4. **Une seule version ACTIVE à la fois** — refuser de démarrer une v1.5 si v1.0 n'est pas COMPLETED
5. **`ContextManager` charge en hiérarchie stricte** — jamais tout le projet en contexte
6. **Tests obligatoires** — `ExecutionLoop` ne marque pas une tâche `done` si les tests échouent
7. **Les fichiers `.workflow/` sont la source de vérité** — Git est synchronisé depuis Workflow, pas l'inverse

---

## 📊 Outils MCP Exposés (Workflow Core)

```
── Phase Projet ──────────────────────────────────────────────
workflow_start_project(description)
workflow_save_discovery(answers)
workflow_propose_features()
workflow_save_features(validated)
workflow_generate_tasks()
workflow_validate_task(taskId, approved)
workflow_set_tech_stack(stack)

── Gestion des Versions ──────────────────────────────────────
workflow_version_list()
workflow_version_create(name, description)
workflow_version_switch(version)         ← bloque si repo non propre
workflow_version_add_task(version, task)
workflow_version_hotfix(name, reason)    ← bloque si repo non propre
workflow_version_complete()

── Contexte & Exécution ──────────────────────────────────────
workflow_get_current_task()
workflow_get_project_context()
workflow_search_codebase(query)
workflow_mark_task_done(taskId)
workflow_mark_task_failed(taskId, reason)
workflow_log_decision(taskId, decision, reason)

── Génération & Audit ────────────────────────────────────────
workflow_doc_generate()
workflow_audit()
workflow_estimate(version)
workflow_onboard()
```

---

## 🚀 Roadmap

### MVP — Workflow Core + CLI de base
- Phase 1 : Foundation complète (ProjectMemory, DecisionsLog, SyncChecker)
- Phase 2 : Les 5 phases projet (génération des fichiers `.workflow/`)
- Phase 3 : ExecutionLoop basique (build_validate + 3 retries + escalade) + DaemonHeartbeat + WatchMode
- Phase 4 : MCP Server complet (branché sur Claude Code immédiatement)
- CLI readline simple pour interagir sans MCP

### V1 — Présence & Intégrations
- Phase 5 : VS Code Extension (sidebar + annotations inline)
- Phase 5 : GitHub/GitLab Integration (PR merged → tâche DONE, CI failed → alerte)
- Phase 5 : `workflow onboard` — onboarding instantané nouveau développeur
- Phase 5 : Détection et résolution de conflits de décisions entre devs

### V1.5 — Robustesse
- Phase 6 : `CodePatcher` (diffs chirurgicaux + fallback AST tree-sitter)
- Phase 6 : `CodeIndexer` (index JSON + variantes LLM + ripgrep)
- Phase 6 : `WorkflowLibrary` (cross-project learning)
- `ContextManager` fin (chargement sélectif avancé)

### V2 — Génération & Audit
- Phase 7 : `workflow doc generate` (README + ARCHITECTURE.md + CHANGELOG auto)
- Phase 7 : `workflow audit` (divergence code/tâches)
- Phase 7 : `workflow estimate` (estimation basée sur historique réel)
- Phase 7 : Tests end-to-end sur cycle complet

### V3 — Écosystème
- Phase 8 : Bot Telegram (notifications + interactions)
- Phase 8 : API REST locale
- Phase 8 : CLI Ink (React terminal)
- Phase 8 : workflow-hub (marketplace de patterns communautaire)
- Protocole `workflow-sync` pour partager `.workflow/` en ligne

---

## 🔄 Suivi de Progression

### Au début de chaque session :
1. **Lire `.claude/PROGRESS.md`** — indique exactement où on s'est arrêté
2. Identifier la phase et la tâche en cours
3. Lire le fichier de tâche correspondant
4. Reprendre exactement là où on s'est arrêté

### Après avoir complété une tâche :
1. Mettre à jour `.claude/PROGRESS.md`
2. Committer le code + `PROGRESS.md` ensemble

📄 **Fichier de suivi** : `.claude/PROGRESS.md`

---

## ⚠️ INSTRUCTION GLOBALE

Les extraits de code dans les fichiers de tâches sont **indicatifs**, pas complets. Leur rôle est d'illustrer la structure et la méthodologie. C'est à Claude Code d'écrire le code complet, fonctionnel et robuste en s'en inspirant.

---

*Workflow est développé par Alister. Le but : construire l'outil qu'on aurait voulu avoir pour construire LinkStream — un agent qui ne perd jamais le fil, version par version, du début jusqu'à la livraison.*
