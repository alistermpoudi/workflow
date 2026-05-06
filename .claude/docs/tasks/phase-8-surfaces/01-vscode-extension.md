# Phase 5 — Tâche 5.1 : Extension VS Code

## Objectif

Créer l'extension VS Code qui intègre Workflow directement dans l'éditeur. C'est l'interface la plus importante pour l'adoption : le développeur passe 8h/jour dans VS Code. Workflow en sidebar devient aussi naturel que l'explorateur de fichiers.

## Dépendances

- Phase 4 (MCP Server) ✅

## Fichiers à Créer

- `src/interfaces/VSCodeExtension/extension.js` [CRÉER] — point d'entrée
- `src/interfaces/VSCodeExtension/WorkflowPanel.js` [CRÉER] — sidebar WebView
- `src/interfaces/VSCodeExtension/InlineDecorations.js` [CRÉER] — annotations inline
- `src/interfaces/VSCodeExtension/package.json` [CRÉER] — manifest extension

## Fonctionnalités

### Sidebar (WorkflowPanel)

```
┌─ WORKFLOW ──────────────────────────────────┐
│                                             │
│  FreelanceApp                               │
│  v1.5 ACTIVE  ████████░░  78%              │
│  9/11 tâches terminées                     │
│                                             │
│  ▶ EN COURS                                │
│  TASK-010 — Export PDF                     │
│  src/services/pdf.service.ts               │
│  src/routes/export.routes.ts               │
│                                             │
│  [▶ Lancer]  [⏸ Pause]  [📋 Détails]      │
│                                             │
│  ── PROCHAINE ──────────────────────────── │
│  TASK-011 — Tests E2E Playwright           │
│                                             │
│  ── QUESTIONS ────────────────────────────  │
│  ⚠️  2 questions en attente               │
│  [Voir →]                                  │
│                                             │
│  ── DECISIONS RÉCENTES ─────────────────── │
│  • PDF via puppeteer (pas jsPDF)           │
│  • Export limité à 50 pages v1.5           │
└─────────────────────────────────────────────┘
```

### Annotations Inline (InlineDecorations)

Sur chaque fichier listé dans `filesToModify` de la tâche courante :

```typescript
// src/services/pdf.service.ts
// ┌ Workflow TASK-010 — ce fichier est en cours
// │ Intent : "Export lisible sur mobile, max 2MB"
// └ Critères : [ ] Génération < 3s  [ ] Format A4
```

Sur les fichiers d'une décision technique :

```typescript
// src/lib/database.ts
// ✓ Workflow : Prisma — décision du 12 mars
//   Raison : migrations plus fiables que TypeORM
```

### Commandes VS Code

```
Workflow: Run Next Task         (Ctrl+Shift+W R)
Workflow: Show Status           (Ctrl+Shift+W S)
Workflow: Open Current Task     (Ctrl+Shift+W T)
Workflow: Answer Questions      (Ctrl+Shift+W Q)
Workflow: Show Briefing         (Ctrl+Shift+W B)
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | La sidebar affiche le nom du projet, la version, et l'avancement | ⬜ |
| 2 | La sidebar affiche la tâche en cours avec ses fichiers | ⬜ |
| 3 | Les fichiers de la tâche courante ont une annotation inline | ⬜ |
| 4 | L'extension se reconnecte automatiquement si le daemon redémarre | ⬜ |
| 5 | Le badge de questions en attente s'actualise en temps réel | ⬜ |
| 6 | `Workflow: Run Next Task` déclenche l'exécution sans quitter VS Code | ⬜ |
