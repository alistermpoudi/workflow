# Phase 2 — Tâche 2.5 : ValidationPhase.py + ArchitecturePhase.py

## Objectif

- `ArchitecturePhase` — définit la stack technique, génère `tech-stack.json`, impose TASK-001 et TASK-002
- `ValidationPhase` — génère les fichiers de tâches pour chaque version et les soumet à validation utilisateur

> **Ordre critique** : `ArchitecturePhase` s'exécute **avant** `ValidationPhase`. Elle produit
> `tech-stack.json` et `design.json` dont `ValidationPhase` a besoin pour générer les tâches
> avec le bon contexte stack et les bons mockups UI.

## Dépendances

- Tâches 2.1 ✅, 2.2 ✅, 2.3 ✅, 2.4 ✅
- Phase 1 ✅

## Fichiers à Créer

- `src/workflow/phases/architecture_phase.py` [CRÉER]
- `src/workflow/phases/validation_phase.py` [CRÉER]

---

## ArchitecturePhase

### Responsabilités

1. Lire `vision.md` et `features.json`
2. Proposer une stack technique via LLM (`role='reasoning'`)
3. Demander validation/correction à l'utilisateur
4. Construire `tech-stack.json` complet (language, framework, database, build_validate, test, allowed_commands)
5. Écrire `tech-stack.json` dans `.workflow/`

### Structure `tech-stack.json`

```json
{
  "language": "python",
  "framework": "fastapi",
  "database": "postgresql",
  "orm": "sqlalchemy",
  "build_validate": "uv run ruff check . && uv run mypy src/",
  "test": "uv run pytest",
  "allowed_commands": [
    "uv run pytest",
    "uv run ruff check .",
    "uv run ruff format .",
    "uv run mypy src/",
    "git status",
    "git diff",
    "git add",
    "git commit",
    "git push"
  ]
}
```

> **`allowed_commands` est une whitelist stricte** — MCPServer ne peut exécuter que les commandes
> listées ici. Aucune exception.

### Implémentation

```python
# src/workflow/phases/architecture_phase.py
import json

from workflow.core.project_memory import ProjectMemory
from workflow.llm.llm_provider import LLMProvider

ARCHITECTURE_PROMPT = """À partir de la vision et des fonctionnalités ci-dessous, propose une stack technique.
Retourne UNIQUEMENT un JSON valide avec cette structure :
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
{features}

Retourne UNIQUEMENT le JSON."""


class ArchitecturePhase:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.memory = ProjectMemory(project_root)
        self.llm = llm
        self.io = io

    async def run(self) -> dict:
        """Exécuter la phase Architecture — retourne {'completed': True} si terminée"""
        vision = await self.memory.get_vision()
        features = await self.memory.get_features()

        if not vision or not features:
            self.io.print_error('Vision ou fonctionnalités manquantes — lance Discovery et Specification d\'abord.')
            return {'completed': False}

        self.io.print_header('Phase Architecture — Définition de la Stack Technique')

        existing_stack = await self.memory.get_tech_stack()
        if existing_stack:
            self.io.print_info('Stack existante détectée.')
            self._display_stack(existing_stack)
            keep = self.io.prompt('Conserver cette stack ? [o/N]', default='o').strip().lower()
            if keep in ('o', 'oui', 'y', 'yes', ''):
                return {'completed': True}

        self.io.print_info('Génération de la stack recommandée...')
        v1_features = features.get('v1.0', [])
        raw = await self.llm.ask(
            ARCHITECTURE_PROMPT.format(
                vision=vision,
                features=json.dumps(v1_features, indent=2, ensure_ascii=False),
            ),
            role='reasoning',
        )

        stack = self._parse_json(raw)
        if not stack:
            self.io.print_error('Le LLM n\'a pas retourné de JSON valide.')
            self.io.print(raw)
            return {'completed': False}

        while True:
            self._display_stack(stack)
            choice = self.io.prompt('\n[v]alider | [c]orriger', default='v').strip().lower()

            if choice in ('v', 'valider', ''):
                break
            elif choice in ('c', 'corriger'):
                correction = self.io.prompt('Décris les changements souhaités (ex: utilise MongoDB à la place de PostgreSQL)')
                raw = await self.llm.ask(
                    ARCHITECTURE_PROMPT.format(
                        vision=vision + f'\n\nCorrections : {correction}',
                        features=json.dumps(v1_features, indent=2, ensure_ascii=False),
                    ),
                    role='reasoning',
                )
                stack = self._parse_json(raw) or stack

        await self.memory.save_tech_stack(stack)
        self.io.print_success('Stack technique enregistrée.')
        return {'completed': True}

    def _parse_json(self, raw: str) -> dict | None:
        try:
            text = raw.strip()
            if text.startswith('```'):
                lines = text.split('\n')
                text = '\n'.join(lines[1:-1]) if lines[-1].strip() == '```' else '\n'.join(lines[1:])
            return json.loads(text)
        except json.JSONDecodeError:
            return None

    def _display_stack(self, stack: dict) -> None:
        self.io.print_section('Stack Technique Proposée')
        for key, value in stack.items():
            if key == 'allowed_commands':
                self.io.print(f'  {key} :')
                for cmd in value:
                    self.io.print(f'    - {cmd}')
            else:
                self.io.print(f'  {key} : {value}')
```

---

## ValidationPhase

### Responsabilités

1. Lire `features.json`, `tech-stack.json`, `design.json`
2. Pour chaque version dans `features.json` (en commençant par v1.0) :
   - Générer les tâches via LLM (`role='reasoning'`)
   - Vérifier la règle de granularité (max 4h, max 3 fichiers)
   - Afficher les tâches proposées et demander validation
   - Écrire les fichiers `TASK-XXX.md` dans `.workflow/versions/vX.Y/tasks/`
3. Garantir que TASK-001 (linter) et TASK-002 (tests) sont les 2 premières tâches de v1.0

### Règle de Granularité

```
1 tâche = max 4h de travail = max 3 fichiers créés/modifiés
Si une fonctionnalité dépasse ces seuils → la découper en sous-tâches
```

### Implémentation

```python
# src/workflow/phases/validation_phase.py
import json

from workflow.core.project_memory import ProjectMemory
from workflow.tools.task_manager import TaskManager
from workflow.llm.llm_provider import LLMProvider
from workflow.llm.prompt_builder import PromptBuilder


class ValidationPhase:
    def __init__(self, project_root: str, llm: LLMProvider, io):
        self.memory = ProjectMemory(project_root)
        self.tasks = TaskManager(project_root)
        self.llm = llm
        self.io = io

    async def run(self) -> dict:
        """Exécuter la phase Validation — retourne {'completed': True} si terminée"""
        features = await self.memory.get_features()
        tech_stack = await self.memory.get_tech_stack()
        design = await self.memory.get_design()

        if not features or not tech_stack:
            self.io.print_error('Fonctionnalités ou stack manquantes.')
            return {'completed': False}

        self.io.print_header('Phase Validation — Génération des Tâches')

        for version in sorted(features.keys()):
            await self._generate_version_tasks(version, features, tech_stack, design)

        return {'completed': True}

    async def _generate_version_tasks(
        self,
        version: str,
        features: dict,
        tech_stack: dict,
        design: dict | None,
    ) -> None:
        self.io.print_section(f'Génération des tâches — {version}')

        existing_ids = await self.tasks.list_task_ids(version)
        if existing_ids:
            self.io.print_info(f'{len(existing_ids)} tâches déjà créées pour {version}.')
            regen = self.io.prompt('Régénérer ? [o/N]', default='n').strip().lower()
            if regen not in ('o', 'oui', 'y', 'yes'):
                return

        self.io.print_info('Génération des tâches en cours...')
        raw = await self.llm.ask(
            PromptBuilder.generate_tasks(
                version=version,
                features=features,
                tech_stack=tech_stack,
                existing_task_ids=existing_ids,
                design=design,
            ),
            role='reasoning',
        )

        task_list = self._parse_task_list(raw)
        if not task_list:
            self.io.print_error('Le LLM n\'a pas retourné de JSON valide.')
            return

        # Vérifier et corriger la granularité
        task_list = self._check_granularity(task_list)

        # Pour v1.0 : garantir TASK-001 et TASK-002 en première position
        if version == 'v1.0':
            task_list = self._ensure_foundation_tasks(task_list, tech_stack)

        self._display_tasks(task_list)

        while True:
            choice = self.io.prompt('\n[v]alider | [c]orriger | [r]egénérer', default='v').strip().lower()

            if choice in ('v', 'valider', ''):
                break
            elif choice in ('r', 'regénérer'):
                raw = await self.llm.ask(
                    PromptBuilder.generate_tasks(version, features, tech_stack, existing_ids, design),
                    role='reasoning',
                )
                task_list = self._parse_task_list(raw) or task_list
                task_list = self._check_granularity(task_list)
                if version == 'v1.0':
                    task_list = self._ensure_foundation_tasks(task_list, tech_stack)
                self._display_tasks(task_list)
            elif choice in ('c', 'corriger'):
                correction = self.io.prompt('Décris les corrections')
                corrected_raw = await self.llm.ask(
                    PromptBuilder.generate_tasks(
                        version,
                        features,
                        tech_stack,
                        existing_ids,
                        design,
                    ) + f'\n\nCorrections demandées : {correction}',
                    role='reasoning',
                )
                task_list = self._parse_task_list(corrected_raw) or task_list
                if version == 'v1.0':
                    task_list = self._ensure_foundation_tasks(task_list, tech_stack)
                self._display_tasks(task_list)

        # Écrire les fichiers TASK-XXX.md
        for task in task_list:
            await self.tasks.save_task(version, task)

        # Initialiser progress.json
        await self.memory.init_version_progress(version, [t['id'] for t in task_list])

        self.io.print_success(f'{len(task_list)} tâches créées pour {version}.')

    def _parse_task_list(self, raw: str) -> list[dict] | None:
        try:
            text = raw.strip()
            if text.startswith('```'):
                lines = text.split('\n')
                text = '\n'.join(lines[1:-1]) if lines[-1].strip() == '```' else '\n'.join(lines[1:])
            result = json.loads(text)
            return result if isinstance(result, list) else None
        except json.JSONDecodeError:
            return None

    def _check_granularity(self, tasks: list[dict]) -> list[dict]:
        """Marquer les tâches qui violent la règle de granularité (log seulement — pas de split auto)"""
        for task in tasks:
            files = task.get('files') or []
            if len(files) > 3:
                self.io.print_warning(
                    f"[{task.get('id', '?')}] {len(files)} fichiers > 3 — envisage de découper cette tâche."
                )
        return tasks

    def _ensure_foundation_tasks(self, tasks: list[dict], tech_stack: dict) -> list[dict]:
        """Garantir que TASK-001 (linter) et TASK-002 (tests) sont en tête de liste pour v1.0"""
        ids = [t.get('id') for t in tasks]

        task_001 = {
            'id': 'TASK-001',
            'title': 'Setup du projet et configuration du linter',
            'context': 'Initialisation de la base du projet avec la stack définie.',
            'user_story': 'EN TANT QUE développeur, JE VEUX un projet initialisé avec linter, AFIN DE garantir la qualité du code dès le départ.',
            'intent': 'Fondation du projet — toutes les tâches suivantes en dépendent.',
            'dependencies': [],
            'files': [],
            'criteria': [
                f'`{tech_stack.get("build_validate", "lint")}` passe sans erreur',
                'Structure de dossiers conforme à la stack',
                'README avec instructions d\'installation',
            ],
        }
        task_002 = {
            'id': 'TASK-002',
            'title': 'Configuration du framework de tests avec smoke test',
            'context': 'Mise en place du framework de tests et premier test de smoke.',
            'user_story': 'EN TANT QUE développeur, JE VEUX un framework de tests fonctionnel, AFIN DE valider chaque fonctionnalité automatiquement.',
            'intent': 'Garantit que toute régression est détectée immédiatement.',
            'dependencies': ['TASK-001'],
            'files': [],
            'criteria': [
                f'`{tech_stack.get("test", "test")}` passe avec au moins 1 test',
                'Test de smoke vérifie que l\'application démarre',
            ],
        }

        # Retirer TASK-001/002 s'ils existent déjà (pour éviter doublons)
        filtered = [t for t in tasks if t.get('id') not in ('TASK-001', 'TASK-002')]
        return [task_001, task_002] + filtered

    def _display_tasks(self, tasks: list[dict]) -> None:
        for task in tasks:
            files = task.get('files') or []
            deps = ', '.join(task.get('dependencies') or []) or 'aucune'
            self.io.print(
                f"\n  [{task.get('id', '?')}] {task.get('title', '')}"
                f"\n    Fichiers : {len(files)} | Dépendances : {deps}"
            )
```

---

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `ArchitecturePhase` génère `tech-stack.json` avec `allowed_commands` | ⬜ |
| 2 | `ArchitecturePhase` propose de conserver une stack existante | ⬜ |
| 3 | `ArchitecturePhase` utilise `role='reasoning'` | ⬜ |
| 4 | `ValidationPhase` lit `design.json` et le passe à `PromptBuilder.generate_tasks()` | ⬜ |
| 5 | `_ensure_foundation_tasks()` garantit TASK-001 et TASK-002 en tête pour v1.0 | ⬜ |
| 6 | `_check_granularity()` avertit si une tâche dépasse 3 fichiers | ⬜ |
| 7 | Les tâches validées sont écrites dans `.workflow/versions/vX.Y/tasks/` | ⬜ |
| 8 | `progress.json` est initialisé avec tous les IDs de tâches en `pending` | ⬜ |
| 9 | La boucle de validation accepte `v` / `c` / `r` pour chaque version | ⬜ |
| 10 | `ValidationPhase` utilise `role='reasoning'` pour tous les appels LLM | ⬜ |
