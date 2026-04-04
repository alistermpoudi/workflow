# Phase 5 — Tâche 5.3 : workflow onboard

## Objectif

Permettre à un nouveau développeur sur le projet de devenir autonome en moins d'une minute. `workflow onboard` lit tout le `.workflow/`, génère un résumé adapté, et lui assigne sa première tâche.

## Dépendances

- Phase 4 (MCP Server) ✅

## Fichiers à Créer

- `src/core/OnboardingManager.js` [CRÉER]
- `tests/unit/OnboardingManager.test.js` [CRÉER]

## Expérience Utilisateur

```bash
git clone https://github.com/marc/client-tracker
cd client-tracker
npm install -g workflow
workflow onboard
```

```
─── Workflow — Onboarding ───────────────────────

Bienvenue sur Client Tracker.
Je lis le projet...  ████████████████  100%

APPLICATION
  SaaS de suivi de projet pour freelances.
  Les clients (PME, non-techniques) voient
  l'avancement sans solliciter le freelance.

STACK
  Next.js 14 + PostgreSQL + Prisma + Resend
  Auth : magic link (pas de mot de passe)
  Upload : Uploadthing v4

ÉTAT ACTUEL
  v1.5 ACTIVE  ████████░░  78%
  9/11 tâches terminées

LES 5 DÉCISIONS CLÉS
  1. Auth par magic link — friction zéro pour clients non-techniques
  2. Uploadthing v4 (pas v3) — breaking change détecté semaine 3
  3. iron-session (pas next-auth) — plus simple pour magic link custom
  4. shadcn/ui (pas MUI) — Tailwind déjà en place
  5. WebSocket reporté v1.5 — incompatible Vercel en v1.0

TA PREMIÈRE TÂCHE
  TASK-010 : Export PDF — puppeteer
  ~3h | 2 fichiers | Dépend de TASK-009 ✅

Pour démarrer : workflow run
─────────────────────────────────────────────────
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `workflow onboard` fonctionne sur un projet cloné sans session précédente | ⬜ |
| 2 | Les 5 décisions sont triées par score d'importance (confidence + scope) | ⬜ |
| 3 | La première tâche suggérée respecte les dépendances (toutes ✅) | ⬜ |
| 4 | L'onboarding prend moins de 30 secondes sur un projet de 20 tâches | ⬜ |
| 5 | Si le projet est en phase Discovery (pas encore de tâches), message adapté | ⬜ |
