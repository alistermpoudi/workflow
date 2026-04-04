# Phase 7 — Tâche 7.2 : workflow audit

## Objectif

Détecter les divergences entre ce que Workflow sait du projet et ce que le code contient réellement. C'est le filet de sécurité qui garantit que `.workflow/` reste synchronisé avec la réalité.

## Types de Divergences Détectées

```
workflow audit

─── Workflow Audit — Client Tracker ────────────

✅ 9 tâches terminées — fichiers présents
⚠️  2 divergences détectées

  [1] Fichier sans tâche associée
      src/lib/socket.ts
      → Créé le 2026-03-15, aucune tâche ne le couvre
      → Action : créer TASK-012 ? (o/n)

  [2] Dépendance non déclarée
      package.json contient "puppeteer" (v21)
      → Non listée dans tech-stack.json
      → Action : ajouter à tech-stack.json ? (o/n)

✅ Stack cohérente (Prisma, Next.js, Resend)
✅ Toutes les décisions HIGH ont un fichier correspondant
─────────────────────────────────────────────────
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | Détecte un fichier `src/` sans tâche associée | ⬜ |
| 2 | Détecte une dépendance dans package.json non dans tech-stack.json | ⬜ |
| 3 | Propose des actions correctives (créer tâche, mettre à jour stack) | ⬜ |
| 4 | `workflow audit --fix` applique les corrections sans confirmation | ⬜ |
| 5 | `workflow audit --json` sort le résultat en JSON (pour CI) | ⬜ |
