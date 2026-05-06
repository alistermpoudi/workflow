# Phase 2 — Tâche 2.1 : LLMProvider.py

## Objectif

Créer l'abstraction LLM avec **LiteLLM** et **routage par rôle**. Chaque appel LLM dans Workflow déclare son rôle ; `LLMProvider` route vers le modèle approprié (Claude Opus pour reasoning, DeepSeek Coder pour code, Haiku pour fast, etc.) avec fallback sur `default_model` en cas d'indisponibilité.

> **Pilier load-bearing #3.** Le LLM qui génère 50 tâches détaillées (`reasoning`) n'est pas le même que celui qui code (`code_generation`), ni que celui qui score 200 décisions (`fast`), ni que celui qui consolide les skills (`curator`), ni que celui qui résume une session (`compression`). Single-LLM partout = soit médiocre, soit insoutenable financièrement.

## Dépendances

- Phase 1 complète ✅

## Fichiers à Créer

- `src/workflow/llm/llm_provider.py` [CRÉER]
- `workflow.config.yaml` [CRÉER — dans le projet utilisateur ou ~/.workflow/]
- `tests/unit/test_llm_provider.py` [CRÉER]

## Les 5 Rôles

| Rôle | Usage typique | Modèle recommandé |
|------|---------------|-------------------|
| `reasoning` | Discovery, Specification, Architecture — pensée structurée longue | `claude-opus-4-7` |
| `code_generation` | ExecutionLoop — génération de patches | `deepseek-coder-v2` ou `claude-sonnet-4-6` |
| `fast` | Scoring décisions, détection contradictions, summaries | `claude-haiku-4-5` |
| `curator` | Consolidation périodique des skills cross-projet | `claude-sonnet-4-6` |
| `compression` | Résumé de session, compression de contexte 3-phases | `claude-haiku-4-5` |

## Configuration — `workflow.config.yaml`

```yaml
# Cherché dans cet ordre :
# 1. ./workflow.config.yaml (projet)
# 2. ~/.workflow/config.yaml (global)

models:
  reasoning: "anthropic/claude-opus-4-7"
  code_generation: "deepseek/deepseek-coder-v2"
  fast: "anthropic/claude-haiku-4-5"
  curator: "anthropic/claude-sonnet-4-6"
  compression: "anthropic/claude-haiku-4-5"

# Fallback si un modèle spécialisé est indisponible
default_model: "anthropic/claude-sonnet-4-6"

# Limites par rôle (optionnel)
limits:
  reasoning:
    max_tokens: 16000
    temperature: 0.3
  code_generation:
    max_tokens: 8000
    temperature: 0.1
  fast:
    max_tokens: 1000
    temperature: 0.0
  curator:
    max_tokens: 4000
    temperature: 0.2
  compression:
    max_tokens: 2000
    temperature: 0.0

# Clés API — peuvent référencer des variables d'environnement
api_keys:
  anthropic: "${ANTHROPIC_API_KEY}"
  deepseek: "${DEEPSEEK_API_KEY}"
```

### Configuration 100% locale (offline)

```yaml
models:
  reasoning: "ollama/qwen2.5:72b"
  code_generation: "ollama/qwen2.5-coder:32b"
  fast: "ollama/llama3.2:3b"
  curator: "ollama/qwen2.5:14b"
  compression: "ollama/llama3.2:3b"
default_model: "ollama/qwen2.5:14b"
```

## Implémentation

```python
# src/workflow/llm/llm_provider.py
import os
from pathlib import Path
import yaml
import litellm
from litellm import acompletion

litellm.set_verbose = False
litellm.suppress_debug_info = True


class LLMProvider:
    def __init__(self, config: dict | None = None):
        self.config = config or {}
        self.models: dict[str, str] = self.config.get('models', {})
        self.limits: dict[str, dict] = self.config.get('limits', {})
        self.default_model: str = self.config.get(
            'default_model', 'anthropic/claude-sonnet-4-6'
        )
        self._setup_api_keys()

    def _setup_api_keys(self):
        for provider, key in self.config.get('api_keys', {}).items():
            if isinstance(key, str) and key.startswith('${') and key.endswith('}'):
                env_var = key[2:-1]
                key = os.environ.get(env_var, '')
            if key:
                os.environ[f'{provider.upper()}_API_KEY'] = key

    def get_model(self, role: str) -> str:
        return self.models.get(role, self.default_model)

    def _get_limits(self, role: str) -> dict:
        defaults = {'max_tokens': 8192, 'temperature': 0.2}
        return {**defaults, **self.limits.get(role, {})}

    async def ask(
        self,
        prompt: str,
        role: str = 'fast',
        system: str | None = None,
        max_tokens: int | None = None,
        **kwargs,
    ) -> str:
        """Appel simple — retourne le texte de la réponse"""
        model = self.get_model(role)
        limits = self._get_limits(role)
        messages = [{'role': 'user', 'content': prompt}]
        if system:
            messages = [{'role': 'system', 'content': system}] + messages

        try:
            response = await acompletion(
                model=model,
                messages=messages,
                max_tokens=max_tokens or limits['max_tokens'],
                temperature=limits.get('temperature', 0.2),
                **kwargs,
            )
            return response.choices[0].message.content

        except Exception:
            # Fallback sur default_model si modèle spécialisé indisponible
            if model != self.default_model:
                response = await acompletion(
                    model=self.default_model,
                    messages=messages,
                    max_tokens=max_tokens or limits['max_tokens'],
                    temperature=limits.get('temperature', 0.2),
                )
                return response.choices[0].message.content
            raise

    async def chat(
        self,
        history: list[dict],
        role: str = 'reasoning',
        system: str | None = None,
    ) -> str:
        """Conversation multi-tours (pour les phases interactives)"""
        model = self.get_model(role)
        limits = self._get_limits(role)
        messages = history.copy()
        if system:
            messages = [{'role': 'system', 'content': system}] + messages

        response = await acompletion(
            model=model,
            messages=messages,
            max_tokens=limits['max_tokens'],
            temperature=limits.get('temperature', 0.2),
        )
        return response.choices[0].message.content

    async def stream(
        self,
        prompt: str,
        role: str = 'fast',
        system: str | None = None,
        on_chunk=None,
    ) -> str:
        """Streaming avec callback par chunk"""
        model = self.get_model(role)
        limits = self._get_limits(role)
        messages = [{'role': 'user', 'content': prompt}]
        if system:
            messages = [{'role': 'system', 'content': system}] + messages

        response = await acompletion(
            model=model,
            messages=messages,
            max_tokens=limits['max_tokens'],
            stream=True,
        )
        full_text = ''
        async for chunk in response:
            delta = chunk.choices[0].delta
            if delta.content:
                full_text += delta.content
                if on_chunk:
                    on_chunk(delta.content)
        return full_text

    @classmethod
    def from_config_file(cls, config_path: str | Path | None = None) -> 'LLMProvider':
        """Charger la config depuis workflow.config.yaml (projet > global)"""
        paths_to_try = [
            Path(config_path) if config_path else None,
            Path.cwd() / 'workflow.config.yaml',
            Path.home() / '.workflow' / 'config.yaml',
        ]
        for path in paths_to_try:
            if path and path.exists():
                with open(path, encoding='utf-8') as f:
                    config = yaml.safe_load(f) or {}
                return cls(config)
        return cls({})
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `ask(prompt, role='code_generation')` utilise le modèle configuré pour ce rôle | ⬜ |
| 2 | `ask(prompt, role='reasoning')` utilise Claude Opus | ⬜ |
| 3 | Si le modèle spécialisé échoue (exception), fallback sur `default_model` | ⬜ |
| 4 | `from_config_file()` charge `./workflow.config.yaml` en priorité, sinon `~/.workflow/config.yaml` | ⬜ |
| 5 | Config 100% locale (ollama) fonctionne sans clé API externe | ⬜ |
| 6 | Variables d'environnement `${VAR}` sont résolues correctement | ⬜ |
| 7 | `limits` par rôle sont appliquées (max_tokens, temperature) | ⬜ |
| 8 | `stream()` appelle `on_chunk` pour chaque chunk reçu | ⬜ |
| 9 | Tests unitaires mockent `litellm.acompletion` | ⬜ |
| 10 | Aucun appel sans rôle déclaré (pas de défaut implicite à `reasoning`) | ⬜ |

## Note d'Architecture

`PromptBuilder` (qui construit les prompts avec contexte injecté) est traité dans la **tâche 2.2** (`02-prompt-builder.md`). La séparation est volontaire : `LLMProvider` est l'abstraction transport ; `PromptBuilder` est la logique métier de construction du prompt. Couplés ailleurs, ils deviendraient un god-object difficile à tester.
