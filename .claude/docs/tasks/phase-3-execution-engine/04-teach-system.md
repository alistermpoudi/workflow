# Phase 3 — Tâche 3.4 : TeachSystem (workflow teach / avoid)

## Objectif

Créer le **système d'enseignement direct** — `workflow teach` et `workflow avoid`. Le dev peut **directement injecter des skills** dans Workflow sans attendre que l'agent les découvre par retry-success. C'est la voie d'apprentissage la plus rapide.

> **Pilier load-bearing #2 (étendu).** Le user_explicit + user_negative complètent les 3 autres sources : auto_retry, project_ingestion, in_flow_correction. Sans cette voie, on attend 6 mois que l'agent découvre des trucs que le dev sait déjà.

## Dépendances

- Tâche 1.5 ✅ (`SkillManager`)
- Tâche 1.9 ✅ (`ContextRegistry`)

## Fichiers à Créer

- `src/workflow/core/teach_system.py` [CRÉER]
- `tests/unit/test_teach_system.py` [CRÉER]

## Confidence `USER_OVERRIDE`

Nouveau niveau ajouté à l'échelle existante :

```
USER_OVERRIDE  (NEW)  ← édicté par le dev, bat tout
HIGH                  ← décision explicite validée
MEDIUM                ← annotation DÉCISION:/RAISON: dans le code
LOW                   ← inféré par observation
```

Quand un skill `USER_OVERRIDE` contredit un skill `HIGH` auto-créé, **`USER_OVERRIDE` gagne**. Le SkillCurator NE PEUT PAS supprimer ni archiver un skill `USER_OVERRIDE` sans confirmation explicite.

## CLI

### `workflow teach`

```bash
# Forme implicite — Workflow détecte le scope (context actif si dans un projet, sinon _global)
workflow teach "go_router toujours pour la nav, jamais Navigator 1.0"

# Forme explicite avec scope
workflow teach --context mobile.flutter "go_router toujours, jamais Navigator 1.0"
workflow teach --context _global "commits format français impératif"

# Forme avec tags pour matching plus fin
workflow teach --tags "auth,jwt" "toujours rotation des refresh tokens 30j"

# Personnel (non partagé via git si dans un projet team)
workflow teach --personal "j'aime les semicolons explicites en JS"
```

### `workflow avoid`

```bash
workflow avoid "ne propose plus de class-based React components"
workflow avoid --context backend.fastapi "pas de @middleware pour l'auth, utiliser Depends"
workflow avoid --personal "pas d'emoji dans les commits"
```

Les anti-patterns sont stockés différemment des skills positifs : ils sont injectés dans le **system prompt** comme section `AVOID:`.

## Stockage

```
~/.workflow/contexts/<context>/
├── skills/                          ← skills positifs (incluant teach)
│   └── teach-go-router-nav.md       ← source: user_explicit, confidence: USER_OVERRIDE
└── avoid.md                         ← anti-patterns en bullets

# Pour les skills personnels (non partageables)
~/.workflow/contexts/<context>/personal/
├── skills/
└── avoid.md
```

> Le dossier `personal/` est dans le `.gitignore` du context si celui-ci est versionné.

## Format `avoid.md`

```markdown
# Anti-patterns — context: mobile.flutter

## React-style class components
- Reason: Workflow avait proposé du class-based, mais Marc préfère functional + hooks partout
- Added: 2026-04-15
- Source: workflow avoid

## Try/except autour de chaque appel DB
- Reason: laisse remonter, le middleware global gère
- Added: 2026-05-02
- Source: workflow avoid
```

Format simple, lisible. Le `LLMContextLoader` extrait les bullets et les injecte en `AVOID:` dans le system prompt.

## Implémentation

```python
# src/workflow/core/teach_system.py
from datetime import datetime, timezone
from pathlib import Path
from typing import Literal

from workflow.core.skill_manager import SkillManager
from workflow.core.context_registry import ContextRegistry, GLOBAL_CONTEXT_NAME


Visibility = Literal['shared', 'personal']


class TeachSystem:
    def __init__(self, registry: ContextRegistry, skills: SkillManager):
        self.registry = registry
        self.skills = skills

    # ─── Teach (positif) ──────────────────────────────────────────────

    def teach(
        self,
        rule: str,
        context: str | None = None,
        tags: list[str] | None = None,
        visibility: Visibility = 'shared',
        title: str | None = None,
    ) -> dict:
        """
        Créer un skill USER_OVERRIDE depuis une règle énoncée par le dev.
        """
        ctx_name = context or self._infer_context()
        ctx = self.registry.get(ctx_name)
        if not ctx:
            raise ValueError(f'Context inexistant : {ctx_name}')

        # Générer un titre court depuis la règle si non fourni
        title = title or self._auto_title(rule)

        skill_data = {
            'name': f'teach-{title}',
            'description': rule[:120],
            'tags': (tags or []) + ['user_explicit'],
            'body': self._format_skill_body(rule),
            'source': 'user_explicit',
            'confidence': 'USER_OVERRIDE',
            'context': ctx_name,
            'visibility': visibility,
        }

        skill_dir = ctx.path / ('personal/skills' if visibility == 'personal' else 'skills')
        skill_dir.mkdir(parents=True, exist_ok=True)

        path = self._write_skill(skill_dir, skill_data)
        return {
            'name': skill_data['name'],
            'context': ctx_name,
            'path': str(path),
            'visibility': visibility,
        }

    # ─── Avoid (négatif) ──────────────────────────────────────────────

    def avoid(
        self,
        rule: str,
        context: str | None = None,
        visibility: Visibility = 'shared',
        reason: str = '',
    ) -> dict:
        """Ajouter un anti-pattern dans avoid.md du context"""
        ctx_name = context or self._infer_context()
        ctx = self.registry.get(ctx_name)
        if not ctx:
            raise ValueError(f'Context inexistant : {ctx_name}')

        avoid_dir = ctx.path / 'personal' if visibility == 'personal' else ctx.path
        avoid_dir.mkdir(parents=True, exist_ok=True)
        avoid_file = avoid_dir / 'avoid.md'

        if not avoid_file.exists():
            avoid_file.write_text(
                f'# Anti-patterns — context: {ctx_name}\n\n', encoding='utf-8'
            )

        entry = (
            f'\n## {self._auto_title(rule).replace("-", " ").capitalize()}\n'
            f'- Rule: {rule}\n'
            f'- Reason: {reason or "—"}\n'
            f'- Added: {datetime.now(timezone.utc).date().isoformat()}\n'
            f'- Source: workflow avoid\n'
        )
        with open(avoid_file, 'a', encoding='utf-8') as f:
            f.write(entry)

        return {'context': ctx_name, 'visibility': visibility, 'rule': rule}

    # ─── Lecture (pour LLMContextLoader) ──────────────────────────────

    def get_active_avoidances(self, active_contexts: list[str]) -> list[str]:
        """Retourner toutes les règles AVOID pour les contexts actifs"""
        rules: list[str] = []
        for ctx_name in active_contexts:
            ctx = self.registry.get(ctx_name)
            if not ctx:
                continue
            for avoid_file in [ctx.path / 'avoid.md', ctx.path / 'personal' / 'avoid.md']:
                if avoid_file.exists():
                    rules.extend(self._parse_avoidances(avoid_file))
        return rules

    @staticmethod
    def _parse_avoidances(avoid_file: Path) -> list[str]:
        """Extraire les bullets `- Rule: ...` du fichier"""
        rules: list[str] = []
        for line in avoid_file.read_text(encoding='utf-8').splitlines():
            line = line.strip()
            if line.startswith('- Rule:'):
                rules.append(line[len('- Rule:'):].strip())
        return rules

    # ─── Helpers ──────────────────────────────────────────────────────

    @staticmethod
    def _infer_context() -> str:
        """
        Inférer le context depuis le projet courant si on est dans un projet,
        sinon retomber sur _global.
        """
        cwd = Path.cwd()
        project_json = cwd / '.workflow' / 'project.json'
        if project_json.exists():
            import json
            try:
                data = json.loads(project_json.read_text())
                # Le plus spécifique des active_contexts
                actives = data.get('active_contexts', [GLOBAL_CONTEXT_NAME])
                return actives[-1] if actives else GLOBAL_CONTEXT_NAME
            except json.JSONDecodeError:
                pass
        return GLOBAL_CONTEXT_NAME

    @staticmethod
    def _auto_title(rule: str) -> str:
        """Slug court depuis la règle"""
        words = [w.lower() for w in rule.split() if len(w) > 3][:5]
        slug = '-'.join(words)
        # Nettoyer chars problématiques
        return ''.join(c for c in slug if c.isalnum() or c == '-')

    @staticmethod
    def _format_skill_body(rule: str) -> str:
        return f"""## Règle (USER_OVERRIDE)

{rule}

## Source

Édictée par le dev via `workflow teach`. Ne pas archiver/supprimer sans confirmation utilisateur explicite.
"""

    def _write_skill(self, skill_dir: Path, skill_data: dict) -> Path:
        import yaml
        path = skill_dir / f'{skill_data["name"]}.md'
        fm = {
            'name': skill_data['name'],
            'description': skill_data['description'],
            'version': '1.0',
            'platforms': ['darwin', 'linux', 'win32'],
            'source': skill_data['source'],
            'confidence': skill_data['confidence'],
            'context': skill_data['context'],
            'visibility': skill_data['visibility'],
            'tags': skill_data.get('tags', []),
            'usage_count': 0,
            'last_used': None,
            'created_at': datetime.now(timezone.utc).date().isoformat(),
        }
        content = (
            '---\n'
            + yaml.dump(fm, allow_unicode=True, default_flow_style=False)
            + '---\n\n'
            + skill_data['body']
        )
        path.write_text(content, encoding='utf-8')
        return path
```

## Intégration

### Avec `SkillCurator` (tâche 3.2)

Le Curator doit **respecter** les skills `USER_OVERRIDE` :

```python
# Dans SkillCurator
def _select_for_archive(self, skills: list[dict]) -> list[str]:
    archived = []
    for s in skills:
        if s.get('confidence') == 'USER_OVERRIDE':
            continue  # Jamais archivé sans confirmation
        # ... reste de la logique
```

### Avec `LLMContextLoader` (Phase 2 renommée)

```python
async def get_task_context(self, version: str, task_id: str) -> dict:
    ...
    # Skills positifs depuis SkillManager
    skill_context = self.skills.get_skills_for_context(task, active_contexts)
    # Anti-patterns depuis TeachSystem
    avoidances = self.teach_system.get_active_avoidances(active_contexts)
    if avoidances:
        avoid_block = '## AVOID — règles édictées par le dev\n' + '\n'.join(
            f'- {a}' for a in avoidances
        )
        skill_context = avoid_block + '\n\n' + skill_context
    ...
```

### CLI (Phase 5)

```python
@app.command()
def teach(
    rule: str,
    context: str = typer.Option(None, '--context', '-c'),
    tags: list[str] = typer.Option([], '--tag', '-t'),
    personal: bool = typer.Option(False, '--personal'),
):
    teach_system.teach(rule, context, tags, 'personal' if personal else 'shared')

@app.command()
def avoid(
    rule: str,
    reason: str = typer.Option('', '--reason', '-r'),
    context: str = typer.Option(None, '--context', '-c'),
    personal: bool = typer.Option(False, '--personal'),
):
    teach_system.avoid(rule, context, 'personal' if personal else 'shared', reason)
```

### MCP (Phase 6)

```
workflow_teach(rule, context, tags, visibility)
workflow_avoid(rule, reason, context, visibility)
workflow_list_teach_rules(context)
workflow_remove_teach_rule(rule_name)
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `teach(rule)` sans context infère depuis `.workflow/project.json#active_contexts` | ⬜ |
| 2 | `teach(rule)` hors d'un projet utilise `_global` | ⬜ |
| 3 | Le skill créé a `confidence: USER_OVERRIDE` et `source: user_explicit` | ⬜ |
| 4 | `--personal` place le skill dans `<context>/personal/skills/` | ⬜ |
| 5 | `avoid(rule)` ajoute une entrée dans `avoid.md` du context | ⬜ |
| 6 | `get_active_avoidances([_global, mobile])` retourne les rules de tous les avoid.md | ⬜ |
| 7 | `SkillCurator` n'archive jamais un skill `USER_OVERRIDE` automatiquement | ⬜ |
| 8 | `_auto_title()` génère un slug propre à partir d'une règle libre | ⬜ |
| 9 | Tests : teach simple, teach with tags, teach personal, avoid, parsing avoid.md | ⬜ |
| 10 | `LLMContextLoader` injecte les avoidances en préfixe `## AVOID` du prompt | ⬜ |

## Notes d'Implémentation

- **Pas d'IA pour l'enseignement** : `workflow teach` est purement déclaratif — le dev sait ce qu'il dit, pas besoin que le LLM "interprète". Plus rapide, plus prévisible.
- **Conflit teach vs auto-skill** : si le dev fait `workflow avoid "X"` alors qu'un skill auto suggère X, le `USER_OVERRIDE` masque le skill auto au prompt-time (sans le supprimer du disque — il pourrait redevenir pertinent dans un autre context).
- **Audit** : tous les `teach`/`avoid` sont versionnés dans git (sauf personal). Permet de voir qui a édicté quoi dans une équipe.
- **Phase 8** : la commande `/teach` Telegram bot wrap simplement ce système.
