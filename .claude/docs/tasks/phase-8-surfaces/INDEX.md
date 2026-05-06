# Phase 8 — Surfaces Tierces (Clients du Protocole MCP)

## Objectif

Ajouter les surfaces non-CLI : **VS Code extension**, **GitHub integration**, **Telegram bot**, **REST API**. Toutes ces surfaces sont des **clients du protocole MCP** (Pilier 6) — elles n'ont aucune logique métier propre, elles formatent et délèguent.

> **Discipline architecturale** : si une nouvelle feature nécessite de modifier *seulement* une de ces surfaces (sans toucher MCP), c'est un signal d'alarme — la logique devrait être dans le core.

## Piliers Adressés

- **Pilier 6 (achevé en pratique)** — toutes les surfaces sont des clients MCP

## Stack Phase 8

```
TypeScript           VS Code extension (obligatoire — VS Code API en TS)
@octokit/rest        GitHub integration (Python via PyGithub)
python-telegram-bot  Telegram bot
FastAPI + uvicorn    REST API locale
MCPClient            Client MCP réutilisable (créé en 8.3)
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 8.1 | [01-vscode-extension.md](01-vscode-extension.md) | VS Code extension — sidebar état projet + annotations inline |
| 8.2 | [02-github-integration.md](02-github-integration.md) | GitHub/GitLab — webhooks, PR mergée → tâche DONE, CI failed → alerte |
| 8.3 | [03-telegram-bot.md](03-telegram-bot.md) | **Telegram bot** + `MCPClient` réutilisable (briefing matin, /next, /done, /curate) |
| 8.4 | [04-rest-api.md](04-rest-api.md) | **REST API locale** (FastAPI) — pour CI/CD, dashboards web, automation tierce |
| 8.5 | [05-team-onboarding.md](05-team-onboarding.md) | Onboarding équipe — gestion multi-utilisateur des décisions et conventions |
| 8.6 | [06-conflict-resolution.md](06-conflict-resolution.md) | Conflits décisions entre devs (équipe partageant `.workflow/`) |
| 8.7 | [07-zero-to-pr.md](07-zero-to-pr.md) | Pipeline "tâche → PR" automatisé end-to-end |

## Dépendances

- Phase 6 complète ✅ (MCP Server)
- Phase 7 (Daemon) recommandée — pour les briefings auto Telegram

## Critères de Sortie de Phase

- [ ] `MCPClient` est utilisable par TelegramBot, RestAPI, et tout futur client
- [ ] VS Code extension affiche la tâche courante + annotations inline
- [ ] VS Code extension parle au MCP (pas accès direct à `.workflow/`)
- [ ] PR mergée → `workflow_mark_task_done` automatiquement via webhook
- [ ] CI failed → notification Telegram + ajout à `.workflow/questions/`
- [ ] `/status`, `/next`, `/done`, `/decisions`, `/skills`, `/curate` fonctionnent en Telegram
- [ ] Briefing matinal envoyé via Telegram à l'heure configurée
- [ ] REST API `GET /project`, `POST /tasks/X/done` répondent correctement
- [ ] REST API exige Bearer token si `api_key` configurée
- [ ] REST API n'expose PAS sur 0.0.0.0 par défaut (localhost only)
- [ ] `workflow onboard` (Phase 10) consomme `workflow_onboard()` MCP dans tous les clients
- [ ] Aucune logique métier dupliquée entre les surfaces — uniquement formatage + appels MCP
