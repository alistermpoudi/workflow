# Phase 2 — Cerveau Cognitif

## Objectif

Construire la couche cognitive : abstraction LLM avec **routage par rôle** (Pilier 3), construction de prompts avec contexte projet injecté, gestionnaire de contexte hiérarchique avec compression Hermes-style, et **CodePatcher chirurgical dès le départ** (Pilier 5 — pas en v1.5).

À la fin de cette phase, Workflow a un cerveau utilisable : il sait penser dans les 5 rôles (`reasoning`, `code_generation`, `fast`, `curator`, `compression`), construit des prompts avec contexte injecté, et applique des modifications de code sans régénérer des fichiers entiers.

## Piliers Adressés

- **Pilier 3** — Multi-LLM par rôle (tâches 2.1 + 2.2)
- **Pilier 5** — CodePatcher chirurgical dès le MVP (tâche 2.4)
- Support du **Pilier 2** (`SkillManager` injecte les skills via `ContextManager`)
- Support du **Pilier 4** (`ContextManager` utilise le `DecisionsGraph` pour scorer)

## Stack Phase 2

```
LiteLLM              Routage multi-modèles par rôle
workflow.config.yaml Configuration des modèles (Claude, DeepSeek, Ollama...)
tree-sitter          AST parsing (CodePatcher fallback)
tree-sitter-languages Grammaires multi-langage
pytest-mock          Mock litellm.acompletion dans les tests
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 2.1 | [01-llm-provider.md](01-llm-provider.md) | `LLMProvider.py` — LiteLLM multi-modèles par rôle + fallback |
| 2.2 | [02-prompt-builder.md](02-prompt-builder.md) | `PromptBuilder.py` — prompts avec contexte projet/skills/décisions injectés |
| 2.3 | [03-llm-context-loader.md](03-llm-context-loader.md) | `LLMContextLoader.py` — hiérarchie stricte + compression 3 phases + scoring décisions + **context-aware** (charge skills/decisions des contexts actifs) |
| 2.4 | [04-code-patcher.md](04-code-patcher.md) | **`CodePatcher.py`** — search/replace blocks + AST fallback (jamais de fichier complet régénéré) |

## Dépendances

- Phase 1 complète ✅

## Critères de Sortie de Phase

- [ ] `LLMProvider.ask(prompt, role='code_generation')` route vers DeepSeek (ou config locale)
- [ ] Si modèle spécialisé indisponible, fallback automatique sur `default_model`
- [ ] Configuration 100% locale (Ollama) fonctionne sans clé API externe
- [ ] `PromptBuilder.generate_code()` impose le format SEARCH/REPLACE pour MODIFIER
- [ ] `PromptBuilder.generate_code()` injecte `task.intent`, skills, décisions scorées
- [ ] `LLMContextLoader` ne charge jamais plus que le niveau nécessaire
- [ ] `LLMContextLoader` charge skills + decisions de TOUTE la chaîne des contexts actifs
- [ ] `LLMContextLoader` injecte les skills pertinents via `SkillManager.search(active_contexts=...)`
- [ ] `LLMContextLoader` injecte les avoidances (`AVOID:`) depuis `TeachSystem.get_active_avoidances()`
- [ ] `LLMContextLoader._load_scored_decisions()` respecte le budget 2000 tokens
- [ ] Skills `USER_OVERRIDE` ont la priorité maximale au scoring (battent HIGH/MEDIUM)
- [ ] La compression 3 phases réduit le contexte sans perdre les informations critiques
- [ ] `CodePatcher.parse_patches()` extrait CRÉER, MODIFIER (search/replace), AST_REPLACE
- [ ] `CodePatcher.apply()` MODIFY échoue avec message actionnable si SEARCH ambigu
- [ ] `CodePatcher` ne touche jamais à un fichier hors `project_root`
