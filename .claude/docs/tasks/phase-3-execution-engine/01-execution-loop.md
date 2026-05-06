# Phase 3 — Tâche 3.1 : ExecutionLoop.py

## Objectif

Implémenter la boucle d'auto-correction. `ExecutionLoop` génère le code via LLM, l'applique sur le système de fichiers, lance `build_validate`, lance les tests, et se corrige automatiquement jusqu'à 3 fois avant d'escalader. **Après un retry réussi, il crée automatiquement un skill** pour que Workflow n'ait jamais à refaire la même correction.

## Dépendances

- Phases 1 et 2 complètes ✅

## Fichiers à Créer

- `src/workflow/tools/execution_loop.py` [CRÉER]

## Constantes

```
MAX_RETRIES = 3
```

## Implémentation

```python
# src/workflow/tools/execution_loop.py
import asyncio
import re
from pathlib import Path
from datetime import datetime, timezone

from workflow.llm.llm_provider import LLMProvider
from workflow.llm.prompt_builder import PromptBuilder
from workflow.core.decisions_log import DecisionsLog
from workflow.core.skill_manager import SkillManager

MAX_RETRIES = 3


class ExecutionLoop:
    def __init__(self, project_root: str, llm: LLMProvider, io, tech_stack: dict):
        self.project_root = Path(project_root)
        self.llm = llm
        self.io = io
        self.tech_stack = tech_stack
        self.decisions = DecisionsLog(project_root)
        self.skills = SkillManager(project_root)

    async def run(self, task_ctx: dict) -> dict:
        """
        Boucle principale : génère → applique → valide → corrige (max 3x).
        Retourne {'success': True} ou {'success': False, 'error': str}.
        """
        task = task_ctx['task']
        relevant_files = task_ctx.get('relevant_files', {})
        relevant_decisions = task_ctx.get('relevant_decisions', [])
        skill_context = task_ctx.get('skill_context', '')

        # Injecter les skills dans le contexte des décisions
        if skill_context:
            self.io.print_info('Skills pertinents chargés.')

        last_error: dict | None = None
        attempts: list[str] = []

        for attempt in range(MAX_RETRIES + 1):
            if attempt == 0:
                prompt = PromptBuilder.generate_code(task, relevant_files, relevant_decisions)
                if skill_context:
                    prompt = f"## Skills Réutilisables\n{skill_context}\n\n---\n\n" + prompt
            else:
                known_fix = self.skills.find_fix_for_error(last_error.get('output', ''))
                prompt = PromptBuilder.generate_code_retry(
                    task, relevant_files, relevant_decisions, last_error, known_fix
                )
                self.io.print_warning(f'Retry {attempt}/{MAX_RETRIES}...')

            self.io.print_info('Génération du code...')
            raw_response = await self.llm.ask(prompt, role='code_generation')
            attempts.append(raw_response)

            # Extraire et enregistrer les décisions techniques annotées
            await self._extract_decisions(task['id'], raw_response)

            # Appliquer les fichiers générés
            applied_files = self._apply_generated_files(raw_response)
            if not applied_files:
                self.io.print_warning('Aucun fichier généré dans la réponse LLM.')

            # Valider avec build_validate
            validate_result = await self._run_command(self.tech_stack.get('build_validate', ''))
            if validate_result['exit_code'] != 0:
                last_error = validate_result
                if attempt < MAX_RETRIES:
                    self.io.print_error(f'build_validate échoué :\n{validate_result["output"][:500]}')
                    continue
                return {'success': False, 'error': validate_result['output'], 'attempts': attempts}

            # Lancer les tests
            test_result = await self._run_command(self.tech_stack.get('test', ''))
            if test_result['exit_code'] != 0:
                last_error = test_result
                if attempt < MAX_RETRIES:
                    self.io.print_error(f'Tests échoués :\n{test_result["output"][:500]}')
                    continue
                return {'success': False, 'error': test_result['output'], 'attempts': attempts}

            # Succès
            if attempt > 0:
                # Retry réussi → créer un skill automatiquement
                await self._maybe_create_skill(task, attempts, last_error)

            self.io.print_success(f'Tâche {task["id"]} validée en {attempt + 1} tentative(s).')
            return {'success': True, 'attempts': attempts}

        return {'success': False, 'error': 'Max retries atteint', 'attempts': attempts}

    def _apply_generated_files(self, raw: str) -> list[str]:
        """Parser la réponse LLM et écrire les fichiers (format ### chemin/fichier.py)"""
        applied = []
        # Pattern : ### chemin/fichier.ext suivi d'un bloc ```
        pattern = re.compile(
            r'^### (.+?)\n```(?:\w+)?\n(.*?)```',
            re.MULTILINE | re.DOTALL,
        )
        for match in pattern.finditer(raw):
            file_path = self.project_root / match.group(1).strip()
            content = match.group(2)
            file_path.parent.mkdir(parents=True, exist_ok=True)
            file_path.write_text(content, encoding='utf-8')
            applied.append(str(file_path))
            self.io.print_info(f'  Écrit : {match.group(1).strip()}')
        return applied

    async def _run_command(self, command: str) -> dict:
        """Exécuter une commande de la whitelist allowed_commands"""
        if not command:
            return {'exit_code': 0, 'output': ''}

        allowed = self.tech_stack.get('allowed_commands', [])
        # Vérifier que la commande (ou un préfixe) est dans la whitelist
        cmd_base = command.split()[0] if command else ''
        if not any(command.startswith(a.split()[0]) for a in allowed):
            return {
                'exit_code': 1,
                'output': f'Commande non autorisée : {command!r} — ajoute-la à allowed_commands.',
            }

        proc = await asyncio.create_subprocess_shell(
            command,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.STDOUT,
            cwd=str(self.project_root),
        )
        stdout, _ = await proc.communicate()
        return {
            'exit_code': proc.returncode,
            'output': stdout.decode('utf-8', errors='replace'),
        }

    async def _extract_decisions(self, task_id: str, raw: str) -> None:
        """Extraire les annotations DÉCISION:/RAISON: et les persister dans DecisionsLog"""
        pattern = re.compile(
            r'DÉCISION:\s*(.+?)\nRAISON:\s*(.+?)(?=\n(?:DÉCISION:|FIX:|$)|\Z)',
            re.DOTALL,
        )
        for match in pattern.finditer(raw):
            decision_text = match.group(1).strip()
            reason_text = match.group(2).strip()
            await self.decisions.add({
                'task_id': task_id,
                'summary': decision_text,
                'reason': reason_text,
                'confidence': 'HIGH',
                'scope': 'task',
                'date': datetime.now(timezone.utc).date().isoformat(),
            })

    async def _maybe_create_skill(
        self, task: dict, attempts: list[str], last_error: dict | None
    ) -> None:
        """Créer automatiquement un skill quand un retry réussit"""
        if not last_error:
            return

        error_summary = last_error.get('output', '')[:200]
        fix_attempt = attempts[-1][:500] if attempts else ''

        skill_data = {
            'name': f"fix-{task['id'].lower()}-retry",
            'description': f"Fix auto-appris lors de {task['id']} : {error_summary[:80]}",
            'version': '1.0',
            'platforms': [],
            'learned_from': task['id'],
            'tags': ['auto-learned', 'retry-fix'],
            'content': f"""## Erreur rencontrée

```
{error_summary}
```

## Fix appliqué

{fix_attempt}
""",
        }
        path = await self.skills.create_skill(skill_data, scope='global')
        self.io.print_info(f'Skill créé : {path}')
```

## Séquence Détaillée

```
ExecutionLoop.run(task_ctx)
  │
  ├─ attempt=0 : PromptBuilder.generate_code()
  │    └─ skill_context injecté en préfixe si disponible
  ├─ LLMProvider.ask(role='code_generation')
  ├─ _extract_decisions() → DecisionsLog.add()
  ├─ _apply_generated_files() → écriture fichiers
  ├─ _run_command(build_validate) → exit_code
  ├─ _run_command(test) → exit_code
  │
  ├─ Si échec → attempt=1,2,3 :
  │    ├─ SkillManager.find_fix_for_error() → known_fix
  │    ├─ PromptBuilder.generate_code_retry() + known_fix
  │    └─ [même séquence]
  │
  └─ Si retry réussi → _maybe_create_skill() → SkillManager.create_skill(scope='global')
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `run()` génère le code avec `role='code_generation'` | ⬜ |
| 2 | `_apply_generated_files()` parse le format `### chemin/fichier.ext` + ` ```lang ` | ⬜ |
| 3 | `_run_command()` vérifie la whitelist `allowed_commands` avant d'exécuter | ⬜ |
| 4 | La boucle s'arrête après `MAX_RETRIES` même si les tests échouent toujours | ⬜ |
| 5 | `_extract_decisions()` persiste les annotations `DÉCISION:/RAISON:` | ⬜ |
| 6 | Après un retry réussi, `_maybe_create_skill()` crée un skill global | ⬜ |
| 7 | Le skill créé contient l'erreur et le fix appliqué | ⬜ |
| 8 | `skill_context` est injecté en préfixe du premier prompt | ⬜ |
| 9 | `run()` retourne `{'success': False, 'error': ..., 'attempts': [...]}` en cas d'échec final | ⬜ |
| 10 | `_run_command` utilise `asyncio.create_subprocess_shell` avec cwd=project_root | ⬜ |
