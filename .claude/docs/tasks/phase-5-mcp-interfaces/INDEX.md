# Phase 5 — Interfaces Supplémentaires (V2)

## Objectif

Étendre Workflow avec des interfaces supplémentaires. Cette phase est distincte du MVP — elle ajoute de la valeur après que le noyau est stable et testé.

## Dépendances

- Phase 4 (MVP) complète ✅

## Tâches

| Tâche | Fichier | Description | Priorité |
|-------|---------|-------------|----------|
| 5.1 | [01-telegram-bot.md](01-telegram-bot.md) | `TelegramBot.js` — notifications + interactions courtes | V2 |
| 5.2 | [02-rest-api.md](02-rest-api.md) | `RestAPI.js` — API REST locale Express | V2 |
| 5.3 | [03-cli-ink.md](03-cli-ink.md) | Migration CLI vers Ink (React terminal) | V2 |
| 5.4 | [04-pipe-cli.md](04-pipe-cli.md) | `PipeCLI.js` — stdin/stdout pipe pour scripts | V2 |

## Notes

- **Telegram** : Utile pour les notifications ("tâche X terminée") et les confirmations rapides. L'interaction code doit rester en CLI — Telegram n'est pas adapté à la revue de code.
- **REST API** : Permet d'intégrer Workflow dans des éditeurs ou outils sans support MCP.
- **CLI Ink** : Migration progressive depuis readline — même interface, meilleure UX.
- **Pipe CLI** : Permet d'utiliser Workflow dans des scripts CI/CD.
