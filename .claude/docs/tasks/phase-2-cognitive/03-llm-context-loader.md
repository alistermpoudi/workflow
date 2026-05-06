# Phase 2 — Tâche 2.3 : LLMContextLoader.py

## Objectif

Créer le `LLMContextLoader.py` avec hiérarchie de chargement stricte et **compression de contexte 3 phases**. Garantit que Workflow ne sature jamais le contexte en chargeant tout le projet d'un coup, et comprime intelligemment quand nécessaire. **Charge les skills + decisions des contexts actifs** (Pilier 7) avec scoring pondéré par spécificité.

> **Renommage critique** : ce module s'appelait `ContextManager` dans le design initial. Renommé en `LLMContextLoader` pour libérer le mot "context" pour le concept utilisateur (Pilier 7 — contexts spécialisés). Ce module *charge le contexte LLM* — c'est exactement ce qu'il fait.

## Dépendances

- Tâche 2.1 ✅ (`LLMProvider`)
- Tâche 2.2 ✅ (`PromptBuilder`)
- Tâche 1.5 ✅ (`SkillManager`)
- Tâche 1.7 ✅ (`DecisionsGraph`)
- Tâche 1.9 ✅ (`ContextRegistry` — pour résoudre la chaîne d'héritage des contexts actifs)

## Fichiers à Créer

- `src/workflow/core/llm_context_loader.py` [CRÉER]
- `tests/unit/test_llm_context_loader.py` [CRÉER]

## Comportement context-aware (Pilier 7)

Le `LLMContextLoader` lit les `active_contexts` de `project.json` et charge skills + decisions de **toute la chaîne d'héritage** :

```
project active_contexts: [_global, mobile, mobile.flutter]
                              │       │         │
                              ▼       ▼         ▼
LLMContextLoader charge :   ~/.workflow/contexts/_global/skills + decisions
                            ~/.workflow/contexts/mobile/skills + decisions
                            ~/.workflow/contexts/mobile.flutter/skills + decisions
                            .workflow/skills (projet) + .workflow/decisions.log

Scoring : skills du context le plus spécifique > skills du général
          USER_OVERRIDE > HIGH > MEDIUM > LOW
```

## Règle Fondamentale

```
Niveau 1 — Système   : toujours chargé (~500 tokens max)
Niveau 2 — Version   : chargé au switch de version
Niveau 3 — Tâche     : chargé au start task
Niveau 4 — On-demand : CodeIndexer — Phase 6
```

**Ne jamais charger tout le projet en contexte.**

## Compression de Contexte 3 Phases (inspirée de Hermes)

Quand la conversation devient trop longue (> 70% du budget tokens), compression automatique :

```
Phase 1 — Pré-compression (sans LLM) :
  Remplacer les anciens outputs longs par des résumés courts
  Ex: "[terminal] pytest → exit 0, 47 tests passed (résumé auto)"

Phase 2 — Protection des zones critiques :
  Protéger la tête (system prompt + premiers échanges)
  Protéger la queue (4 derniers tours)

Phase 3 — Summarisation structurée (LLM role='compression') :
  Résumer le milieu en JSON structuré :
  { active_task, completed_actions, pending_asks, key_decisions }
  Anti-boucle : si gain < 10% sur 2 compressions consécutives → arrêter
```

## Scoring de Pertinence des Décisions

```
Score(décision, tâche) =
  similarité_mots_clés(décision.summary, tâche.title + tâche.intent)  [0–1]
  × poids_récence(décision.date)   [0.5 | 0.8 | 1.0]
  × poids_scope(décision.scope)    [0.3 | 0.8 | 1.0]
  × (confidence == 'HIGH' ? 1.0 : 0.7)

Seuil minimum : 0.4
Budget tokens décisions : 2000
```

## Implémentation

```python
# src/workflow/core/llm_context_loader.py
from workflow.core.project_memory import ProjectMemory
from workflow.tools.task_manager import TaskManager
from workflow.core.decisions_log import DecisionsLog
from workflow.core.skill_manager import SkillManager
from workflow.tools.filesystem import FileSystem
from workflow.llm.llm_provider import LLMProvider


class LLMContextLoader:
    def __init__(self, project_root: str, llm: LLMProvider | None = None):
        self.memory = ProjectMemory(project_root)
        self.tasks = TaskManager(project_root)
        self.decisions = DecisionsLog(project_root)
        self.skills = SkillManager(project_root)
        self.fs = FileSystem(project_root)
        self.llm = llm

        # Cache en mémoire pour éviter les relectures répétées
        self._system_context: dict | None = None
        self._version_context: dict | None = None
        self._active_version: str | None = None
        self._compression_count: int = 0
        self._last_compression_savings: float = 1.0

    async def get_system_context(self) -> dict:
        """Niveau 1 — Contexte système (toujours chargé, mis en cache)"""
        if self._system_context:
            return self._system_context

        summary = await self.memory.get_project_summary()
        tech_stack = await self.memory.get_tech_stack()

        self._system_context = {
            'project': summary,
            'tech_stack': {
                'language': tech_stack.get('language'),
                'framework': tech_stack.get('framework'),
                'database': tech_stack.get('database'),
            } if tech_stack else None,
        }
        return self._system_context

    async def get_version_context(self, version: str) -> dict:
        """Niveau 2 — Contexte version (chargé au switch)"""
        if self._active_version == version and self._version_context:
            return self._version_context

        meta = await self.memory.get_version_meta(version)
        progress = await self.memory.get_progress(version)

        self._version_context = {
            'meta': meta,
            'done_tasks': progress['done'],
            'pending_tasks': progress['pending'],
            'failed_tasks': progress.get('failed', []),
            'deferred_tasks': progress.get('deferred', []),
        }
        self._active_version = version
        return self._version_context

    async def get_task_context(self, version: str, task_id: str) -> dict:
        """Niveau 3 — Contexte tâche (chargé au start task)"""
        task = await self.tasks.get_task(version, task_id)
        if not task:
            raise ValueError(f'Tâche {task_id} introuvable dans la version {version}')

        # Lire sélectivement les fichiers mentionnés dans la tâche
        relevant_files = await self.fs.read_selective(task.get('files_to_modify') or [])

        # Décisions scorées par pertinence
        relevant_decisions = await self._load_scored_decisions(task)

        # Skills pertinents (synchrone — lecture fichiers locaux)
        skill_context = self.skills.get_skills_for_context(task)

        return {
            'task': task,
            'relevant_files': relevant_files,
            'relevant_decisions': relevant_decisions,
            'skill_context': skill_context,
        }

    async def _load_scored_decisions(
        self, task: dict, token_budget: int = 2000, min_score: float = 0.4
    ) -> list[dict]:
        """Charger les décisions avec scoring de pertinence"""
        all_decisions = await self.decisions.get_all()
        if not all_decisions:
            return []

        scored = [
            {**d, 'score': self._score_decision(d, task)}
            for d in all_decisions
        ]
        filtered = sorted(
            [d for d in scored if d['score'] >= min_score],
            key=lambda x: x['score'],
            reverse=True,
        )

        # Appliquer le budget tokens (approximation : ~4 chars/token)
        result = []
        used_tokens = 0
        for d in filtered:
            tokens = len(d.get('summary', '') + d.get('reason', '')) // 4
            if used_tokens + tokens > token_budget:
                break
            result.append(d)
            used_tokens += tokens
        return result

    def _score_decision(self, decision: dict, task: dict) -> float:
        """Calculer le score de pertinence d'une décision par rapport à une tâche"""
        search_text = ' '.join(filter(None, [
            task.get('title', ''),
            task.get('intent', ''),
            task.get('context', ''),
        ])).lower()
        decision_text = (decision.get('summary') or '').lower()

        # Similarité Jaccard simplifiée
        task_words = {w for w in search_text.split() if len(w) > 3}
        dec_words = {w for w in decision_text.split() if len(w) > 3}
        intersection = len(task_words & dec_words)
        union = len(task_words | dec_words)
        similarity = intersection / union if union > 0 else 0

        # Récence
        from datetime import date as date_cls
        try:
            age_days = (date_cls.today() - date_cls.fromisoformat(decision.get('date', '2000-01-01'))).days
        except ValueError:
            age_days = 9999
        recency = 1.0 if age_days < 7 else (0.8 if age_days < 30 else 0.5)

        # Scope
        scope = decision.get('scope', 'global')
        if scope == 'global':
            scope_weight = 1.0
        elif scope and scope.lower() in (task.get('context') or '').lower():
            scope_weight = 0.8
        else:
            scope_weight = 0.3

        # Confiance
        confidence_weight = 1.0 if decision.get('confidence') == 'HIGH' else 0.7

        return similarity * recency * scope_weight * confidence_weight

    async def build_llm_context(self, options: dict | None = None) -> dict:
        """Construire le contexte complet pour un appel LLM"""
        options = options or {}
        ctx: dict = {}
        ctx['system'] = await self.get_system_context()

        if options.get('version'):
            ctx['version'] = await self.get_version_context(options['version'])

        if options.get('task_id') and options.get('version'):
            ctx['task'] = await self.get_task_context(options['version'], options['task_id'])

        return ctx

    def invalidate_cache(self):
        """Invalider le cache (après un switch de version ou update projet)"""
        self._system_context = None
        self._version_context = None
        self._active_version = None

    # ─── Compression de contexte 3 phases ────────────────────────────────────

    async def compress_if_needed(
        self, conversation_history: list[dict], token_budget: int = 100_000
    ) -> list[dict]:
        """Comprimer le contexte si > 70% du budget tokens"""
        estimated = sum(len(m.get('content', '')) // 4 for m in conversation_history)
        if estimated < token_budget * 0.7:
            return conversation_history
        return await self._compress_3phase(conversation_history, token_budget)

    async def _compress_3phase(
        self, history: list[dict], budget: int
    ) -> list[dict]:
        if not history or self.llm is None:
            return history

        # Phase 1 — Pré-compression sans LLM
        history = self._precompress(history)

        # Identifier les zones
        head = history[:2]
        tail = history[-4:]
        middle = history[2:-4]
        if not middle:
            return history

        # Phase 3 — Summariser le milieu via LLM
        middle_text = '\n'.join(
            f"[{m.get('role', '?')}]: {m.get('content', '')[:400]}"
            for m in middle
        )
        summary_prompt = f"""Résume ces échanges en JSON strict (sans markdown autour) :
{{
  "active_task": "tâche en cours",
  "completed_actions": ["action 1", "action 2"],
  "pending_asks": ["question en attente"],
  "key_decisions": ["décision technique prise"]
}}

Échanges :
{middle_text}

Retourne UNIQUEMENT le JSON."""

        summary_json = await self.llm.ask(summary_prompt, role='compression')
        summary_msg = {
            'role': 'system',
            'content': f'[RÉSUMÉ DE SESSION]\n{summary_json}',
        }

        compressed = head + [summary_msg] + tail

        # Anti-boucle : vérifier le gain
        original_tokens = sum(len(m.get('content', '')) // 4 for m in history)
        compressed_tokens = sum(len(m.get('content', '')) // 4 for m in compressed)
        savings = (original_tokens - compressed_tokens) / max(original_tokens, 1)

        if savings < 0.1 and self._last_compression_savings < 0.1:
            return history  # Deux compressions consécutives < 10% → arrêter

        self._last_compression_savings = savings
        self._compression_count += 1
        return compressed

    def _precompress(self, history: list[dict]) -> list[dict]:
        """Phase 1 — Remplacer les outputs longs par des résumés courts"""
        result = []
        for msg in history:
            content = msg.get('content', '')
            if '[terminal]' in content and len(content) > 500:
                lines = content.split('\n')
                summary = f'[terminal] {lines[0]} → {len(lines)} lignes (résumé automatique)'
                result.append({**msg, 'content': summary})
            else:
                result.append(msg)
        return result
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `get_system_context()` met en cache et ne relit pas les fichiers deux fois | ⬜ |
| 2 | `get_version_context()` ne charge pas le contenu des tâches, uniquement les IDs | ⬜ |
| 3 | `get_task_context()` lit sélectivement les fichiers listés dans la tâche | ⬜ |
| 4 | `get_task_context()` injecte les skills pertinents via `SkillManager` | ⬜ |
| 5 | `build_llm_context({ 'version': v })` ne charge pas le niveau tâche si non demandé | ⬜ |
| 6 | `invalidate_cache()` force un rechargement complet | ⬜ |
| 7 | `_score_decision()` retourne un score plus élevé pour mots-clés communs | ⬜ |
| 8 | `_load_scored_decisions()` respecte le budget 2000 tokens | ⬜ |
| 9 | `compress_if_needed()` ne comprime pas si < 70% du budget | ⬜ |
| 10 | `_compress_3phase()` protège head (2 msgs) et tail (4 msgs) | ⬜ |
| 11 | Anti-boucle : arrête si 2 compressions consécutives < 10% de gain | ⬜ |
| 12 | Tests unitaires vérifient que les niveaux sont chargés indépendamment | ⬜ |
