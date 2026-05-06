# Phase 2 — Tâche 2.2 : PromptBuilder.py

## Objectif

Créer `PromptBuilder.py` — la logique de construction des prompts. Chaque méthode statique produit un prompt formaté pour une tâche spécifique (Discovery, génération de tâches, génération de code, retry sur erreur, etc.) en injectant systématiquement le contexte projet, les skills pertinents, et les décisions scorées.

> Séparé de `LLMProvider` volontairement : transport (LLM) vs métier (prompt). Permet de tester les prompts sans appels LLM, et de re-formuler les prompts sans toucher l'abstraction transport.

## Dépendances

- Tâche 2.1 ✅ (`LLMProvider`)

## Fichiers à Créer

- `src/workflow/llm/prompt_builder.py` [CRÉER]
- `tests/unit/test_prompt_builder.py` [CRÉER]

## Principes de Construction

1. **Contexte injecté systématiquement** — nom projet, stack, version active, intent de la tâche
2. **Skills en préfixe** — quand pertinents, injectés AVANT les instructions (ils orientent le raisonnement)
3. **Décisions scorées** — incluses uniquement si pertinentes (scoring `ContextManager`)
4. **Format de sortie strict** — pour `generate_code`, format imposé pour parsing fiable
5. **Annotations métier** — `DÉCISION:/RAISON:`, `FIX:`, `SKILL:` pour extraction post-hoc

## Implémentation

```python
# src/workflow/llm/prompt_builder.py
import json


class PromptBuilder:

    @staticmethod
    def system_prompt(project_context: dict) -> str:
        return f"""Tu es Workflow, un agent de code expert.
Tu travailles sur le projet "{project_context.get('name', '')}" \
({project_context.get('language', '?')}/{project_context.get('framework', '?')}).
Version active : {project_context.get('current_version') or 'aucune'}.

Règles absolues :
- Consulte toujours les décisions techniques passées avant de proposer une solution.
- Respecte la stack définie : {json.dumps(project_context)}.
- Une tâche = 1 PR mergeable atomiquement avec ses tests.
- Jamais de commande shell non listée dans allowed_commands.
- Jamais de régénération de fichier complet — utilise le format patch (search/replace blocks)."""

    # ─── Phase Discovery ──────────────────────────────────────────────

    @staticmethod
    def discovery(partial_vision: str) -> str:
        return f"""L'utilisateur décrit son idée. Pose 3 à 5 questions ciblées pour clarifier :
- Le problème résolu
- Les utilisateurs cibles
- Les fonctionnalités essentielles vs. optionnelles
- Les contraintes techniques éventuelles
- Le style de design (minimaliste, material, glassmorphism, neomorphisme, brutaliste, etc.)

Idée initiale :
{partial_vision}

Pose tes questions de manière concise et directe."""

    # ─── Phase Specification ──────────────────────────────────────────

    @staticmethod
    def spec_features(vision: str, existing_features: dict | None = None) -> str:
        existing = (
            f'\nFonctionnalités existantes à affiner :\n{json.dumps(existing_features, indent=2, ensure_ascii=False)}'
            if existing_features else ''
        )
        return f"""À partir de cette vision produit, génère une liste de fonctionnalités structurée par version.
Retourne un JSON valide avec la structure :
{{"v1.0": [{{"id": "F001", "name": "...", "description": "...", "priority": "HIGH|MEDIUM|LOW", "intent": "pourquoi l'utilisateur veut vraiment cette fonctionnalité"}}], "v1.5": [...]}}

Vision :
{vision}
{existing}

Retourne UNIQUEMENT le JSON, sans explication."""

    # ─── Phase Architecture ───────────────────────────────────────────

    @staticmethod
    def architecture(vision: str, features: dict) -> str:
        return f"""À partir de la vision et des fonctionnalités, propose une stack technique.
Retourne UNIQUEMENT un JSON valide :
{{
  "language": "...",
  "framework": "...",
  "database": "...",
  "orm": "... (optionnel)",
  "build_validate": "commande de lint/typecheck",
  "test": "commande de test",
  "allowed_commands": ["cmd1", "cmd2", ...]
}}

Vision :
{vision}

Fonctionnalités v1.0 :
{json.dumps(features.get('v1.0', []), indent=2, ensure_ascii=False)}

Retourne UNIQUEMENT le JSON."""

    # ─── Phase Validation : génération de tâches ──────────────────────

    @staticmethod
    def generate_tasks(
        version: str,
        features: dict,
        tech_stack: dict,
        existing_task_ids: list[str],
        design: dict | None = None,
    ) -> str:
        design_instruction = ''
        if design:
            design_instruction = f"""
DESIGN STYLE : {design.get('styleLabel', design.get('style', 'minimaliste'))}
Palette : {design.get('colorScheme', 'clair')}
Références : {', '.join(design.get('references', []))}
Notes : {design.get('customNotes') or 'aucune'}

Pour chaque tâche avec interface utilisateur, inclure une section ## Mockup UI avec un dessin ASCII
représentant l'écran dans le style {design.get('styleLabel', 'minimaliste')} défini ci-dessus."""

        return f"""Génère les fichiers de tâches pour la version {version}.

Fonctionnalités à implémenter :
{json.dumps(features.get(version, []), indent=2, ensure_ascii=False)}

Stack technique :
{json.dumps(tech_stack, indent=2, ensure_ascii=False)}

Tâches déjà créées : {', '.join(existing_task_ids) or 'aucune'}
{design_instruction}

RÈGLES OBLIGATOIRES :
1. Si c'est la v1.0, les deux premières tâches doivent être TASK-001 (setup + linter) et TASK-002 (tests + smoke test).
2. Chaque tâche = 1 PR mergeable atomiquement avec ses tests.
3. Si une fonctionnalité dépasse cette atomicité, découpe-la en sous-tâches.
4. Chaque tâche doit être auto-suffisante (contexte, user story, intent, dépendances, fichiers, critères).

Retourne un tableau JSON :
[{{"id": "TASK-XXX", "title": "...", "context": "...", "user_story": "...", "intent": "...", "dependencies": [...], "files": [{{"path": "...", "action": "CRÉER|MODIFIER"}}], "criteria": [...], "scope": "auth|data-layer|ui|...", "tags": [...]}}]

Retourne UNIQUEMENT le JSON."""

    # ─── Phase Execution : génération de code (PATCH FORMAT) ──────────

    @staticmethod
    def generate_code(
        task: dict,
        relevant_files: dict,
        relevant_decisions: list[dict],
        skill_context: str = '',
    ) -> str:
        decisions_text = (
            '\nDécisions techniques précédentes pertinentes :\n'
            + '\n'.join(f"- {d.get('summary', '')}" for d in relevant_decisions)
            if relevant_decisions else ''
        )
        files_text = '\n'.join(
            f"\n### {path}\n```\n{content}\n```"
            for path, content in relevant_files.items()
            if content is not None
        )
        intent_text = (
            f"\n## Intent (pourquoi cette fonctionnalité)\n{task['intent']}"
            if task.get('intent') else ''
        )
        files_list = '\n'.join(
            f"- {f['path']} [{f['action']}]"
            for f in (task.get('files') or [])
        )
        criteria_list = '\n'.join(
            f'- {c}' for c in (task.get('criteria') or [])
        )
        skill_section = (
            f"## Skills Réutilisables (patterns connus)\n{skill_context}\n\n---\n\n"
            if skill_context else ''
        )

        return f"""{skill_section}Implémente la tâche suivante :

# {task['id']} : {task.get('title', '')}

## User Story
{task.get('user_story', '')}
{intent_text}

## Fichiers à créer/modifier
{files_list}

## Critères d'acceptation
{criteria_list}
{decisions_text}

## Fichiers existants (état actuel)
{files_text or '(aucun fichier existant pour cette tâche)'}

Si tu prends une décision technique importante, annote-la ainsi :
DÉCISION: <la décision>
RAISON: <la justification>

Si tu appliques un fix à une erreur connue, annote-le ainsi :
FIX: <description courte du fix appliqué>

## Format de sortie OBLIGATOIRE — PATCH FORMAT (jamais de fichier complet régénéré)

Pour CRÉER un fichier nouveau, utilise :
### CRÉER chemin/vers/nouveau.py
```python
[contenu complet du nouveau fichier]
```

Pour MODIFIER un fichier existant, utilise des SEARCH/REPLACE blocks :
### MODIFIER chemin/vers/existant.py
<<<<<<< SEARCH
[bloc exact à chercher dans le fichier — au moins 3 lignes pour éviter ambiguïtés]
=======
[bloc qui remplace]
>>>>>>> REPLACE

Tu peux empiler plusieurs SEARCH/REPLACE blocks dans le même fichier.
Ne JAMAIS régénérer un fichier entier pour une modification ponctuelle.

Commence directement par les blocs."""

    @staticmethod
    def generate_code_retry(
        task: dict,
        relevant_files: dict,
        relevant_decisions: list[dict],
        last_error: dict,
        known_fix: str | None = None,
        skill_context: str = '',
    ) -> str:
        base = PromptBuilder.generate_code(
            task, relevant_files, relevant_decisions, skill_context
        )
        error_section = (
            f"\n\n---\nTa tentative précédente a échoué avec cette erreur :\n"
            f"```\n{last_error.get('output', '')[:2000]}\n```"
        )
        fix_section = (
            f'\nUn fix connu pour ce type d\'erreur est : {known_fix}'
            if known_fix else ''
        )
        return base + error_section + fix_section + (
            '\n\nCorrige le code en tenant compte de cette erreur. '
            'Ne régénère QUE les blocs SEARCH/REPLACE nécessaires au fix.'
        )

    # ─── Curator : consolidation de skills ────────────────────────────

    @staticmethod
    def curator_consolidate(skills: list[dict]) -> str:
        skills_text = '\n\n'.join(
            f"### {s['name']}\n{s.get('description', '')}\n{s.get('content', '')[:500]}"
            for s in skills
        )
        return f"""Tu es le Curator de Workflow. Consolide ces skills accumulés en patterns réutilisables.

Skills bruts :
{skills_text}

Tâches à effectuer :
1. Identifier les doublons → marquer pour suppression
2. Identifier les patterns récurrents (≥2 skills similaires) → consolider en un skill généralisé
3. Identifier les skills à promouvoir au global (utilisés ≥3 fois) → marquer global
4. Identifier les skills obsolètes (jamais utilisés depuis 90+ jours) → marquer pour archivage

Retourne UNIQUEMENT un JSON :
{{
  "delete": ["skill-name-1", ...],
  "consolidate": [{{"new_name": "...", "merge_from": [...], "description": "...", "content": "..."}}],
  "promote_global": [...],
  "archive": [...]
}}"""

    # ─── Compression : résumé de session ──────────────────────────────

    @staticmethod
    def compress_session(middle_messages_text: str) -> str:
        return f"""Résume ces échanges en JSON strict (sans markdown autour) :
{{
  "active_task": "tâche en cours",
  "completed_actions": ["action 1", "action 2"],
  "pending_asks": ["question en attente"],
  "key_decisions": ["décision technique prise"]
}}

Échanges :
{middle_messages_text}

Retourne UNIQUEMENT le JSON."""
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `system_prompt()` injecte nom + stack + version active | ⬜ |
| 2 | `generate_code()` injecte `task.intent` avant les critères | ⬜ |
| 3 | `generate_code()` exige le format SEARCH/REPLACE pour MODIFIER | ⬜ |
| 4 | `generate_code()` injecte `skill_context` en préfixe si fourni | ⬜ |
| 5 | `generate_code()` inclut les décisions pertinentes scorées | ⬜ |
| 6 | `generate_code_retry()` inclut `last_error['output']` tronqué à 2000 chars | ⬜ |
| 7 | `generate_code_retry()` inclut `known_fix` si fourni par SkillManager | ⬜ |
| 8 | `generate_tasks()` injecte le design style si fourni | ⬜ |
| 9 | `generate_tasks()` exige les champs scope + tags pour activer le DecisionsGraph | ⬜ |
| 10 | `curator_consolidate()` retourne un format JSON parsable pour SkillCurator | ⬜ |
