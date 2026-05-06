# Phase 3 — Tâche 3.7 : ExecutionLoop Extensions

## Objectif

Étendre `ExecutionLoop` avec 4 nouvelles capacités intégrées dans la boucle existante :

1. **TDD Systématique** — génère les tests AVANT le code
2. **Security Review** — scan de sécurité post-génération
3. **Code Review Architecturale** — vérifie le respect du `decisions.log`
4. **Dependency Intelligence** — détecte les dépendances problématiques avant application

Toutes s'intègrent dans `ExecutionLoop.run()` sans en changer la signature.

## Dépendances

- Tâche 3.1 ✅ (`ExecutionLoop` de base)

## Fichier à Modifier

- `src/workflow/tools/execution_loop.py` [MODIFIER]

---

## 1. TDD Systématique

### Concept

```
Mode normal  : génère code → applique → lance tests
Mode TDD     : génère tests → applique → lance tests (tous échouent) 
               → génère code → applique → lance tests (tous passent)
```

Les tests sont générés directement depuis les `criteria` de la tâche — chaque critère devient une fonction de test. Le LLM code ensuite avec un objectif précis : faire passer CES assertions.

### Code

```python
# Ajouter dans ExecutionLoop

TDD_TEST_PROMPT = """Génère UNIQUEMENT le fichier de tests pour cette tâche.
Transforme chaque critère d'acceptation en une fonction de test.

Tâche : {task_id} — {title}
Critères d'acceptation :
{criteria}

Stack de test : {test_command}
Fichiers qui seront créés/modifiés : {files}

RÈGLES :
- Un test par critère d'acceptation
- Tests indépendants, sans état partagé
- Noms de tests explicites (test_should_xxx / test_when_xxx)
- Utilise des mocks pour les dépendances externes
- Tous les tests doivent ÉCHOUER au premier run (le code n'existe pas encore)

Format de réponse : ### tests/{task_id_lower}.py suivi du code."""

async def run_tdd(self, task_ctx: dict) -> dict:
    """Boucle TDD : génère tests → rouge → génère code → vert"""
    task = task_ctx['task']
    relevant_files = task_ctx.get('relevant_files', {})
    relevant_decisions = task_ctx.get('relevant_decisions', [])

    self.io.print_info('[TDD] Génération des tests...')

    # Étape 1 : générer les tests
    criteria_text = '\n'.join(f'- {c}' for c in (task.get('criteria') or []))
    files_text = '\n'.join(
        f.get('path', str(f)) if isinstance(f, dict) else str(f)
        for f in (task.get('files') or [])
    )
    test_prompt = TDD_TEST_PROMPT.format(
        task_id=task['id'],
        title=task.get('title', ''),
        criteria=criteria_text,
        test_command=self.tech_stack.get('test', 'pytest'),
        files=files_text,
    )
    test_code = await self.llm.ask(test_prompt, role='code_generation')
    test_files = self._apply_generated_files(test_code)

    if not test_files:
        self.io.print_warning('[TDD] Aucun fichier de test généré — mode normal activé.')
        return await self.run(task_ctx)

    # Étape 2 : lancer les tests (on s'attend à RED)
    test_result = await self._run_command(self.tech_stack.get('test', ''))
    if test_result['exit_code'] == 0:
        self.io.print_warning('[TDD] Les tests passent déjà sans code — vérifie les assertions.')

    self.io.print_info(f'[TDD] Phase rouge ✓ — {len(test_files)} fichier(s) de test créé(s).')

    # Étape 3 : générer le code pour faire passer les tests
    tdd_context = {
        **task_ctx,
        'tdd_test_output': test_result.get('output', ''),
        'tdd_test_files': test_code,
    }
    return await self.run(tdd_context, tdd_mode=True)
```

Modifier `run()` pour accepter `tdd_mode=False` et injecter le contexte TDD dans le prompt :

```python
async def run(self, task_ctx: dict, tdd_mode: bool = False) -> dict:
    # ...existing code...
    if tdd_mode and task_ctx.get('tdd_test_files'):
        prompt = (
            f"Les tests suivants ont été générés depuis les critères d'acceptation :\n\n"
            f"```python\n{task_ctx['tdd_test_files']}\n```\n\n"
            f"Génère le code qui fait passer ces tests.\n\n---\n\n" + prompt
        )
    # ...
```

---

## 2. Security Review

### Concept

Après `_apply_generated_files()`, avant de valider, un appel LLM rapide (Haiku) analyse le diff pour les failles OWASP top 10 les plus courantes. Si une faille critique est détectée → bloc + retry forcé avec l'erreur de sécurité.

### Code

```python
SECURITY_REVIEW_PROMPT = """Tu es un expert sécurité. Analyse ce code pour les failles de sécurité.

Code généré :
{code}

Vérifie UNIQUEMENT :
1. Secrets/tokens/passwords hardcodés dans le code
2. Requêtes SQL sans paramétrage (injection SQL)
3. Entrées utilisateur utilisées sans validation/sanitisation
4. Commandes shell construites avec des inputs utilisateur (injection de commande)
5. Chemins de fichiers construits avec des inputs utilisateur (path traversal)

Retourne une liste JSON (vide = aucune faille) :
[{{"severity": "critical|high|medium", "type": "hardcoded_secret|sql_injection|...", "location": "fichier:ligne", "message": "..."}}]

Retourne UNIQUEMENT le JSON. Sois concis et précis — seulement les vraies failles."""

async def _security_review(self, generated_code: str) -> list[dict]:
    """Scanner le code généré pour les failles de sécurité."""
    if not generated_code.strip():
        return []

    raw = await self.llm.ask(
        SECURITY_REVIEW_PROMPT.format(code=generated_code[:4000]),
        role='fast',
    )
    try:
        text = raw.strip()
        if text.startswith('```'):
            lines = text.split('\n')
            text = '\n'.join(lines[1:-1])
        issues = json.loads(text) or []
        return [i for i in issues if isinstance(i, dict)]
    except (json.JSONDecodeError, ValueError):
        return []
```

Intégration dans `run()` :

```python
# Après _apply_generated_files(), avant _run_command(build_validate)
security_issues = await self._security_review(raw_response)
critical_issues = [i for i in security_issues if i.get('severity') == 'critical']
if critical_issues:
    for issue in critical_issues:
        self.io.print_error(f'[Security] {issue["type"]} — {issue["message"]}')
    if attempt < MAX_RETRIES:
        last_error = {
            'output': 'FAILLES DE SÉCURITÉ CRITIQUES DÉTECTÉES:\n'
                      + '\n'.join(f'- {i["type"]}: {i["message"]}' for i in critical_issues)
        }
        continue
elif security_issues:
    for issue in security_issues:
        self.io.print_warning(f'[Security] {issue["type"]} — {issue["message"]}')
```

---

## 3. Code Review Architecturale

### Concept

Après génération du code, vérifier que le code respecte les décisions architecturales enregistrées dans `decisions.log`. Exemples de violations détectées :

- Utilisation de `requests` alors que `httpx` a été choisi (DECISION-003)
- `TypeORM` au lieu de `Prisma` choisi
- `bcrypt` au lieu de `argon2` décidé

### Code

```python
ARCHITECTURE_REVIEW_PROMPT = """Tu es un code reviewer. Vérifie que ce code respecte les décisions architecturales du projet.

Décisions architecturales prises :
{decisions}

Code généré :
{code}

Vérifie si le code contredit une des décisions listées.
Retourne une liste JSON (vide = tout est conforme) :
[{{"decision_violated": "résumé de la décision violée", "found_in_code": "ce qui a été trouvé", "location": "fichier"}}]

Retourne UNIQUEMENT le JSON."""

async def _architecture_review(
    self, generated_code: str, relevant_decisions: list[dict]
) -> list[dict]:
    """Vérifier que le code respecte les décisions architecturales."""
    if not relevant_decisions or not generated_code.strip():
        return []

    decisions_text = '\n'.join(
        f'- {d.get("summary", "")} (raison: {d.get("reason", "")})' 
        for d in relevant_decisions[:10]
    )

    raw = await self.llm.ask(
        ARCHITECTURE_REVIEW_PROMPT.format(
            decisions=decisions_text,
            code=generated_code[:3000],
        ),
        role='fast',
    )
    try:
        text = raw.strip()
        if text.startswith('```'):
            lines = text.split('\n')
            text = '\n'.join(lines[1:-1])
        return json.loads(text) or []
    except (json.JSONDecodeError, ValueError):
        return []
```

Intégration dans `run()` après `_apply_generated_files()` :

```python
arch_issues = await self._architecture_review(raw_response, relevant_decisions)
if arch_issues:
    for issue in arch_issues:
        self.io.print_warning(
            f'[Archi] Violation : {issue["decision_violated"]} → {issue["found_in_code"]}'
        )
    if attempt < MAX_RETRIES:
        last_error = {
            'output': 'VIOLATIONS ARCHITECTURALES:\n'
                      + '\n'.join(
                          f'- {i["decision_violated"]}: {i["found_in_code"]}'
                          for i in arch_issues
                      )
        }
        continue
```

---

## 4. Dependency Intelligence

### Concept

Avant d'appliquer le code généré, scanner les imports/dépendances introduits et vérifier :
- Lib déjà dans le projet → utiliser celle-là plutôt qu'en ajouter une autre
- Lib non dans `allowed_commands` → avertir
- Patterns connus de duplication (ex: `requests` + `httpx` dans le même projet)

### Code

```python
DEPENDENCY_CHECK_PROMPT = """Analyse ce code et liste les nouvelles dépendances introduites.

Stack actuelle : {current_deps}
Code généré :
{code}

Identifie :
1. Nouvelles imports/dépendances absentes de la stack actuelle
2. Doublons : une dépendance similaire existe déjà dans la stack (ex: requests vs httpx)

Retourne JSON :
{{
  "new_deps": ["dep1", "dep2"],
  "duplicates": [{{"introduced": "requests", "existing": "httpx", "reason": "même usage HTTP"}}]
}}"""

async def _check_dependencies(self, generated_code: str) -> dict:
    """Détecter les nouvelles dépendances et doublons potentiels."""
    tech_stack = self.tech_stack
    current_deps = json.dumps({
        'language': tech_stack.get('language'),
        'framework': tech_stack.get('framework'),
        'dependencies': tech_stack.get('dependencies', []),
    })

    raw = await self.llm.ask(
        DEPENDENCY_CHECK_PROMPT.format(
            current_deps=current_deps,
            code=generated_code[:2000],
        ),
        role='fast',
    )
    try:
        text = raw.strip()
        if text.startswith('```'):
            lines = text.split('\n')
            text = '\n'.join(lines[1:-1])
        return json.loads(text) or {}
    except (json.JSONDecodeError, ValueError):
        return {}
```

Intégration dans `run()` avant `_apply_generated_files()` :

```python
dep_check = await self._check_dependencies(raw_response)
for dup in dep_check.get('duplicates', []):
    self.io.print_warning(
        f'[Deps] Doublon potentiel : {dup["introduced"]} — '
        f'{dup["existing"]} fait déjà la même chose ({dup["reason"]})'
    )
    # Pas de blocage — avertissement seulement, le dev décide
```

---

## Séquence Complète d'un Cycle ExecutionLoop (avec extensions)

```
ExecutionLoop.run(task_ctx)
  │
  ├─ [TDD] génère tests → applique → run tests (RED) → si ok passer à code
  │
  ├─ LLM génère code (role='code_generation')
  │
  ├─ [Dependency Intelligence] scan imports → warning si doublon
  │
  ├─ apply_generated_files()
  │
  ├─ [Security Review] scan failles → si critique → last_error → retry
  │
  ├─ [Architecture Review] vérifie decisions.log → si violation → last_error → retry
  │
  ├─ [Design Review] vérifie tokens design system → si hardcoded colors → retry
  │
  ├─ _run_command(build_validate) → si échec → retry
  │
  └─ _run_command(test) → si échec → retry
```

---

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `run_tdd()` génère le fichier de test avant le code | ⬜ |
| 2 | `run_tdd()` vérifie que les tests échouent (phase rouge) avant de générer le code | ⬜ |
| 3 | `_security_review()` bloque et force retry sur issues `critical` | ⬜ |
| 4 | `_security_review()` affiche un warning (sans bloquer) sur issues `high` et `medium` | ⬜ |
| 5 | `_architecture_review()` détecte une lib non conforme aux décisions | ⬜ |
| 6 | `_architecture_review()` force un retry avec le message de violation | ⬜ |
| 7 | `_check_dependencies()` n'affiche qu'un warning (pas de blocage) | ⬜ |
| 8 | Tous les checks utilisent `role='fast'` (Haiku) pour minimiser le coût | ⬜ |
| 9 | La séquence complète reste dans la limite `MAX_RETRIES` | ⬜ |
| 10 | Les extensions sont désactivables via des flags (`tdd=False`, `security_review=False`, etc.) | ⬜ |
