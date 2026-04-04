# Phase 7 — Génération & Audit (V2)

## Objectif

Faire de Workflow la source de vérité absolue du projet — capable de générer automatiquement la documentation, détecter les divergences entre code et plan, et estimer les versions futures sur base de l'historique réel.

## Dépendances

- Phase 5 (Présence & Intégrations) complète ✅

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 7.1 | [01-doc-generate.md](01-doc-generate.md) | `workflow doc generate` — README + ARCHITECTURE.md + CHANGELOG auto |
| 7.2 | [02-audit.md](02-audit.md) | `workflow audit` — détection divergences code/tâches |
| 7.3 | [03-estimate.md](03-estimate.md) | `workflow estimate` — estimation basée sur historique réel |
| 7.4 | [04-tests-e2e.md](04-tests-e2e.md) | Tests end-to-end sur cycle complet init → version complete |

## Critères de Sortie de Phase

- [ ] `workflow doc generate` produit un README.md lisible depuis vision.md + features.json
- [ ] `workflow doc generate` produit ARCHITECTURE.md depuis decisions-graph.json
- [ ] `workflow doc generate` produit CHANGELOG.md depuis l'historique des versions
- [ ] `workflow audit` détecte un fichier créé sans tâche associée
- [ ] `workflow audit` détecte une dépendance dans package.json non déclarée dans tech-stack.json
- [ ] `workflow estimate` produit une estimation cohérente basée sur les tâches passées
- [ ] Tests e2e couvrent le cycle complet sur un projet exemple
