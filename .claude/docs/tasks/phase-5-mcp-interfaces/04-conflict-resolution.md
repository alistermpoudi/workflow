# Phase 5 — Tâche 5.4 : Résolution de Conflits de Décisions

## Objectif

Quand deux développeurs travaillent sur le même projet et enregistrent des décisions contradictoires, Workflow les détecte et bloque les tâches concernées jusqu'à résolution explicite.

## Dépendances

- Tâche 1.5 ✅ (DecisionsLog + decisions-graph.json)
- Tâche 5.3 ✅ (OnboardingManager)

## Fichiers à Créer

- `src/core/ConflictResolver.js` [CRÉER]

## Scénario

```
Dev A (Marc) log le 10 mars :
  "On utilise Zustand pour le state management"
  → DECISION-012 enregistrée dans decisions-graph.json

Dev B (Sarah) pull et log le 11 mars :
  "On utilise Redux Toolkit pour le state management"
  → DECISION-015 détectée comme CONTRADICTS DECISION-012

Workflow (au prochain workflow start des deux) :
  ⚠️  Conflit de décision détecté

  DECISION-012 (Marc, 10 mars) : Zustand
  DECISION-015 (Sarah, 11 mars) : Redux Toolkit

  Les tâches TASK-008 et TASK-009 sont bloquées
  jusqu'à résolution.

  Qui tranche ? [Marc / Sarah / Discussion requise]
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | Un conflit CONTRADICTS dans decisions-graph.json bloque les tâches impactées | ⬜ |
| 2 | Le conflit est affiché au démarrage des deux développeurs concernés | ⬜ |
| 3 | La résolution crée une décision SUPERSEDES qui débloque les tâches | ⬜ |
| 4 | Les tâches non impactées par le conflit ne sont pas bloquées | ⬜ |
