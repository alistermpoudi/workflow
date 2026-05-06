# Phase 7 — Tâche 7.3 : workflow estimate

## Objectif

Générer des estimations de versions futures basées sur le temps réel mesuré sur les tâches passées. Fini les estimations au doigt mouillé — Workflow mesure combien de temps chaque type de tâche prend réellement sur CE projet avec CE développeur.

## Expérience

```bash
workflow estimate v2.0
```

```
─── Estimation v2.0 — Client Tracker ───────────

Basé sur 11 tâches complétées en v1.0 + v1.5 :

  Temps réel moyen par type de tâche :
  • Setup/config    : 1h45 (estimé 2h  — -15min)
  • API routes      : 3h20 (estimé 3h  — +20min)
  • Frontend page   : 4h10 (estimé 4h  — +10min)
  • Auth/sécurité   : 5h30 (estimé 4h  — +90min)
  • Tests           : 2h00 (estimé 1h  — +60min)

  v2.0 contient 8 tâches :
  • 2 tâches API routes      → 2 × 3h20  =  6h40
  • 3 tâches Frontend        → 3 × 4h10  = 12h30
  • 1 tâche Auth             → 1 × 5h30  =  5h30
  • 2 tâches Tests           → 2 × 2h00  =  4h00

  Estimation réaliste : 28h40
  (vs estimation initiale de Workflow : 24h)

  ⚠️  Les tâches Auth prennent +37% de plus
     qu'estimé sur ce projet. Budget en conséquence.
─────────────────────────────────────────────────
```

## Mesure du Temps Réel

Le temps est mesuré depuis les commits Git :
- `git log --format="%ai %s" -- TASK-XXX.md` → timestamps de début/fin

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | Les temps réels sont calculés depuis l'historique git | ⬜ |
| 2 | L'estimation distingue les types de tâches | ⬜ |
| 3 | Les dépassements systématiques sont signalés | ⬜ |
| 4 | Fonctionne dès 3 tâches terminées (minimum statistique) | ⬜ |
