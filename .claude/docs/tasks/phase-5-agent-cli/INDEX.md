# Phase 5 — Agent & CLI

## Objectif

Construire le **Workflow Agent autonome** + la CLI humaine. C'est la première surface d'interaction réellement utilisable. À la fin de cette phase, l'utilisateur peut faire `workflow init`, `workflow start`, `workflow run`, `workflow status` depuis son terminal et voir le projet vivre.

## Piliers Adressés

- Pilier 6 (préparation) — la CLI n'est qu'une surface qui parle au core ; en Phase 6, MCP devient la surface primaire et la CLI bascule en client MCP local.

## Stack Phase 5

```
typer                CLI déclarative
rich                 Affichage panels, couleurs, progress
asyncio              Boucle async
WorkflowAgent        Orchestrateur — câble PhaseManager + ExecutionLoop
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 5.1 | [01-cli.md](01-cli.md) | `CLI.py` — interface typer + rich + `RichIO` (panels, couleurs, prompts) |
| 5.2 | [02-workflow-agent.md](02-workflow-agent.md) | `WorkflowAgent.py` — orchestrateur principal, câble PhaseManager / ExecutionLoop / SyncChecker / Starter Mode |
| 5.3 | [03-in-flow-correction.md](03-in-flow-correction.md) | **`InFlowCorrector.py`** — Ctrl+T interrupt pendant l'agent → skill USER_OVERRIDE → retry (Pilier 2 — source `in_flow_correction`) |

## Dépendances

- Phases 1-4 complètes ✅

## Critères de Sortie de Phase

- [ ] `workflow init "Nom"` initialise un `.workflow/` complet avec validation protocole
- [ ] `workflow start` lance/reprend les phases (full mode)
- [ ] `workflow start --quick "description"` déclenche le Starter Mode (saut à ACTIVE)
- [ ] `workflow run` exécute la prochaine tâche prête via `ExecutionPhase` + `ExecutionLoop`
- [ ] `workflow status` affiche l'état du projet (phase, version, done/pending)
- [ ] `workflow onboard` génère le briefing d'onboarding (Phase 10)
- [ ] `RichIO` affiche les résultats avec rich panels et couleurs
- [ ] `WorkflowAgent` orchestre `PhaseManager` et `ExecutionPhase` dans une boucle async
- [ ] `WorkflowAgent` au boot vérifie `SyncChecker` et alerte sur les drifts
- [ ] `WorkflowAgent` charge `LLMProvider.from_config_file()` au démarrage
- [ ] `InFlowCorrector.signal_interrupt()` capture Ctrl+T (SIGINFO sur macOS)
- [ ] Pendant `ExecutionLoop`, l'interruption est consommée AVANT le retry suivant
- [ ] La correction crée un skill `USER_OVERRIDE` dans le context le plus spécifique
- [ ] Le retry suivant inclut automatiquement le nouveau skill via `LLMContextLoader`
- [ ] `workflow teach`, `workflow avoid`, `workflow learn-from` exposés en commandes CLI
