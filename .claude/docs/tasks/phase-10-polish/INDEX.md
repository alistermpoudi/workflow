# Phase 10 — Polish & Différenciateurs

## Objectif

Faire de Workflow la **source de vérité absolue du projet** — capable de générer automatiquement la documentation, détecter les divergences entre code et plan, estimer les versions futures sur base de l'historique réel, et onboarder un nouveau dev en 30 secondes.

Ces 4 commandes sont les **différenciateurs visibles** de Workflow vs les autres agents : aucun outil actuel ne les offre.

## Piliers Adressés

- Met en valeur tous les piliers déjà construits (vision unifiée, mémoire institutionnelle, decisions actives, skills accumulés)

## Stack Phase 10

```
LLMProvider          role='reasoning' pour génération doc
                     role='fast' pour synthèse onboarding
ProjectMemory + DecisionsLog + DecisionsGraph + SkillManager
TaskManager          (pour audit, estimate)
GitManager           (pour audit, estimate basés sur historique réel)
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 10.1 | [01-doc-generate.md](01-doc-generate.md) | `workflow doc generate` — README + ARCHITECTURE.md + CHANGELOG auto |
| 10.2 | [02-audit.md](02-audit.md) | `workflow audit` — détecte divergences code/tâches, contradictions de décisions |
| 10.3 | [03-estimate.md](03-estimate.md) | `workflow estimate` — basé sur l'historique git réel des tâches précédentes |
| 10.4 | [04-onboard.md](04-onboard.md) | **`workflow onboard`** — onboarding instantané d'un nouveau dev en 30 secondes |

## Dépendances

- Phases 1-9 complètes ✅
- Recommandé : ≥1 vrai projet livré avec Workflow pour calibrer estimate

## Critères de Sortie de Phase

- [ ] `workflow doc generate` produit un README cohérent avec vision + features + stack
- [ ] `workflow doc generate` produit un ARCHITECTURE.md basé sur les décisions HIGH-confidence
- [ ] `workflow doc generate` met à jour CHANGELOG.md à chaque version COMPLETED
- [ ] `workflow audit` détecte les fichiers modifiés sans tâche associée
- [ ] `workflow audit` détecte les tâches DONE sans modifications réelles dans le code
- [ ] `workflow audit` liste les contradictions actives via `DecisionsGraph`
- [ ] `workflow estimate v2.0` retourne une estimation basée sur la durée médiane réelle des tâches précédentes
- [ ] `workflow estimate` ajuste selon le scope (UI vs backend) et la complexité (fichiers à toucher)
- [ ] `workflow onboard` génère un briefing complet en < 5s sur projet de 24 tâches
- [ ] `workflow onboard` lit toutes les sources `.workflow/` en parallèle (asyncio.gather)
- [ ] `workflow onboard` propose la prochaine tâche prête (dépendances satisfaites)
- [ ] `workflow onboard` alerte sur les contradictions actives et tâches reportées 2×
