# Phase 5 — Présence & Intégrations (V1)

## Objectif

Rendre Workflow présent partout où le développeur travaille, et connecté aux outils qu'il utilise déjà. À l'issue de cette phase, Workflow est dans la sidebar VS Code, branché sur GitHub, et permet à un nouveau développeur de s'onboarder en 30 secondes.

## Dépendances

- Phase 4 (MVP) complète ✅

## Tâches

| Tâche | Fichier | Description | Priorité |
|-------|---------|-------------|----------|
| 5.1 | [01-vscode-extension.md](01-vscode-extension.md) | Extension VS Code — sidebar état projet + annotations inline | V1 |
| 5.2 | [02-github-integration.md](02-github-integration.md) | GitHub/GitLab webhooks — PR merged → tâche DONE, CI failed → alerte | V1 |
| 5.3 | [03-team-onboarding.md](03-team-onboarding.md) | `workflow onboard` — onboarding instantané nouveau développeur | V1 |
| 5.4 | [04-conflict-resolution.md](04-conflict-resolution.md) | Détection et résolution de conflits de décisions entre développeurs | V1 |

## Critères de Sortie de Phase

- [ ] L'extension VS Code affiche l'état du projet en sidebar
- [ ] Les fichiers en cours de tâche sont annotés inline dans VS Code
- [ ] Une PR mergée sur GitHub marque automatiquement la tâche correspondante comme DONE
- [ ] Un CI qui casse crée une notification + analyse automatique
- [ ] `workflow onboard` permet à un nouveau dev d'être autonome en < 1 minute
- [ ] Un conflit de décision entre deux devs est détecté et bloque la tâche concernée

## Notes Stratégiques

La VS Code extension est la priorité absolue de cette phase. C'est l'interface où le développeur passe 8h/jour. Workflow dans la sidebar devient aussi naturel que l'explorateur de fichiers — c'est ce qui crée l'addiction.

L'intégration GitHub est le deuxième vecteur d'adoption : quand Workflow sait automatiquement qu'une PR est mergée, le développeur n'a jamais besoin de mettre à jour manuellement l'état d'avancement.
