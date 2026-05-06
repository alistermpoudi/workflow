# Phase 7 — Tâche 7.1 : workflow doc generate

## Objectif

Générer automatiquement la documentation du projet depuis les fichiers `.workflow/`. Workflow devient la source de vérité : la documentation est toujours à jour parce qu'elle est générée depuis les données, pas écrite manuellement.

## Fichiers Générés

### README.md
Depuis `vision.md` + `features.json` + `tech-stack.json`
```markdown
# Client Tracker

Application web de suivi de projet pour freelances.
[contenu de vision.md]

## Fonctionnalités v1.0
[liste depuis features.json]

## Stack
[depuis tech-stack.json avec justifications]

## Installation
[depuis tech-stack.json#setup_commands]
```

### ARCHITECTURE.md
Depuis `decisions-graph.json`
```markdown
# Architecture — Décisions Techniques

## Authentification
**Magic link via Resend** (DECISION-004, 12 mars)
Raison : clients non-techniques, friction zéro.
Scope : global | Révisable : non

## Base de données
**PostgreSQL + Prisma** (DECISION-003, 10 mars)
...
```

### CHANGELOG.md
Depuis `versions/*/meta.json` + `versions/*/progress.json`
```markdown
# Changelog

## v1.5 (en cours)
### Ajouté
- Export PDF (TASK-010)
- WebSocket temps réel (TASK-011)

## v1.0 (complété — 2026-03-28)
### Ajouté
- Auth magic link
- Dashboard freelance
...
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | README.md généré contient la vision et les fonctionnalités | ⬜ |
| 2 | ARCHITECTURE.md liste toutes les décisions HIGH confidence | ⬜ |
| 3 | CHANGELOG.md liste toutes les versions avec leurs tâches | ⬜ |
| 4 | `workflow doc generate --dry-run` affiche sans écrire | ⬜ |
| 5 | Les fichiers existants sont mis à jour, pas écrasés brutalement | ⬜ |
