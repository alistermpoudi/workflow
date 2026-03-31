# Phase 2 — Les 5 Phases Projet

## Objectif

Implémenter les 5 phases qui transforment une idée en fichiers `.workflow/` prêts à être exécutés. C'est la partie "cerveau projet" de Workflow — là où l'IA intervient pour la première fois.

À l'issue de cette phase, Workflow peut accompagner un utilisateur de "j'ai une idée" jusqu'à "j'ai un dossier de tâches validées prêtes à coder".

## Dépendances

- Phase 1 complète ✅

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 2.1 | [01-llm-provider.md](01-llm-provider.md) | `LLMProvider.js` + `PromptBuilder.js` — abstraction Claude API |
| 2.2 | [02-context-manager.md](02-context-manager.md) | `ContextManager.js` — hiérarchie de chargement stricte |
| 2.3 | [03-phase-manager.md](03-phase-manager.md) | `PhaseManager.js` — orchestration des 5 phases |
| 2.4 | [04-discovery-spec.md](04-discovery-spec.md) | `DiscoveryPhase.js` + `SpecificationPhase.js` |
| 2.5 | [05-validation-arch.md](05-validation-arch.md) | `ValidationPhase.js` + `ArchitecturePhase.js` (génération tâches + granularité) |
| 2.6 | [06-execution-phase.md](06-execution-phase.md) | `ExecutionPhase.js` — orchestration d'une tâche (sans ExecutionLoop) |

## Critères de Sortie de Phase

- [ ] Workflow peut conduire un utilisateur de la description initiale jusqu'aux tâches générées
- [ ] Les tâches générées respectent la règle de granularité (max 4h, max 3 fichiers)
- [ ] TASK-001 (linter) et TASK-002 (tests) sont toujours les premières tâches d'une v1.0
- [ ] `ContextManager` ne charge jamais plus que le niveau nécessaire
- [ ] Les prompts LLM incluent toujours le contexte projet (pas de question hors contexte)
