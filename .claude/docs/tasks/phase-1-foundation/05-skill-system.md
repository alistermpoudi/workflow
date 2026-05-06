# Phase 1 — Tâche 1.0 : SkillManager.py (Système de Skills)

## Objectif

Créer `SkillManager.py` — le gestionnaire du système de skills cross-projet. Les skills sont des fichiers Markdown qui documentent des patterns et solutions réutilisables. `ExecutionLoop` en crée automatiquement quand il résout un problème difficile (Phase 3). `ContextManager` les injecte dans le prompt LLM avant chaque tâche.

C'est le mécanisme d'**apprentissage cumulatif** de Workflow — il devient plus intelligent à chaque projet.

## Dépendances

Aucune — opérations fichiers pures, pas de LLM dans ce module.

> Le **Curator** (consolidation LLM des skills) est implémenté en Phase 3 (tâche 3.4).

## Fichiers à Créer / Modifier

- `src/workflow/core/skill_manager.py` [CRÉER]
- `tests/unit/test_skill_manager.py` [CRÉER]

## Structure des Skills

```
~/.workflow/skills/          # Skills globaux (cross-projet, partagés entre tous les projets)
  fix-cannot-find-module.md
  prisma-migration.md
  nextjs-auth-setup.md

.workflow/skills/            # Skills locaux (spécifiques au projet courant)
  auth-fix-jwt-expiry.md
  custom-deploy-pattern.md
```

## Format `SKILL.md`

```markdown
---
name: fix-cannot-find-module
description: Résoudre l'erreur "Cannot find module" / ImportError
version: 1.0
platforms: [darwin, linux, win32]
learned_from: TASK-003
learned_at: 2025-03-15
tags: [imports, modules, python, esm]
usage_count: 0
last_used: null
---

## Problème

Cannot find module '...' ou `ImportError: No module named '...'`

## Solution

1. Vérifier que le module est dans `pyproject.toml` dependencies
2. Relancer `uv sync`
3. Vérifier les imports relatifs (`.` vs chemin absolu)

## Contexte

Stack : Python / Node.js ESM
Erreur typique : `ModuleNotFoundError: No module named 'workflow'`
```

## Implémentation

```python
# src/workflow/core/skill_manager.py
from pathlib import Path
from datetime import date as date_cls
import yaml


GLOBAL_SKILLS_DIR = Path.home() / '.workflow' / 'skills'


class SkillManager:
    def __init__(self, project_root: str):
        self.project_root = Path(project_root)
        self.local_skills_dir = self.project_root / '.workflow' / 'skills'
        self.global_skills_dir = GLOBAL_SKILLS_DIR

    def _parse_frontmatter(self, content: str) -> tuple[dict, str]:
        """Parser le frontmatter YAML d'un fichier skill"""
        if not content.startswith('---'):
            return {}, content
        try:
            end = content.index('---', 3)
            fm = yaml.safe_load(content[3:end]) or {}
            body = content[end + 3:].strip()
            return fm, body
        except (ValueError, yaml.YAMLError):
            return {}, content

    def _all_skill_files(self) -> list[Path]:
        """Lister tous les fichiers skill (global + local)"""
        files: list[Path] = []
        for skill_dir in [self.global_skills_dir, self.local_skills_dir]:
            if skill_dir.exists():
                files.extend(skill_dir.glob('*.md'))
        return files

    def search(self, query: str, max_results: int = 5) -> list[dict]:
        """Recherche par mots-clés dans les skills (synchrone — lecture de fichiers locaux)"""
        query_words = {w.lower() for w in query.split() if len(w) > 3}
        results: list[dict] = []

        for skill_file in self._all_skill_files():
            try:
                content = skill_file.read_text(encoding='utf-8')
            except OSError:
                continue

            fm, body = self._parse_frontmatter(content)
            skill_text = ' '.join([
                fm.get('name', ''),
                fm.get('description', ''),
                ' '.join(fm.get('tags', [])),
                body,
            ]).lower()

            score = sum(1 for w in query_words if w in skill_text)
            if score > 0:
                results.append({
                    'name': fm.get('name', skill_file.stem),
                    'description': fm.get('description', ''),
                    'content': body,
                    'score': score,
                    'path': str(skill_file),
                    'is_global': str(self.global_skills_dir) in str(skill_file),
                })

        results.sort(key=lambda x: x['score'], reverse=True)
        return results[:max_results]

    def create_skill(self, skill_data: dict, scope: str = 'local') -> Path:
        """Créer un nouveau fichier skill"""
        skill_dir = self.global_skills_dir if scope == 'global' else self.local_skills_dir
        skill_dir.mkdir(parents=True, exist_ok=True)

        name = skill_data['name'].replace(' ', '-').lower()
        skill_path = skill_dir / f'{name}.md'

        fm = {
            'name': name,
            'description': skill_data.get('description', ''),
            'version': '1.0',
            'platforms': skill_data.get('platforms', ['darwin', 'linux', 'win32']),
            'learned_from': skill_data.get('learned_from', ''),
            'learned_at': date_cls.today().isoformat(),
            'tags': skill_data.get('tags', []),
            'usage_count': 0,
            'last_used': None,
        }

        content = f"---\n{yaml.dump(fm, allow_unicode=True, default_flow_style=False)}---\n\n{skill_data.get('body', '')}\n"
        skill_path.write_text(content, encoding='utf-8')
        return skill_path

    def record_usage(self, skill_path: str):
        """Enregistrer une utilisation d'un skill (incrémente usage_count)"""
        path = Path(skill_path)
        if not path.exists():
            return
        content = path.read_text(encoding='utf-8')
        fm, body = self._parse_frontmatter(content)
        fm['usage_count'] = fm.get('usage_count', 0) + 1
        fm['last_used'] = date_cls.today().isoformat()
        updated = f"---\n{yaml.dump(fm, allow_unicode=True, default_flow_style=False)}---\n\n{body}\n"
        path.write_text(updated, encoding='utf-8')

    def get_skills_for_context(self, task: dict) -> str:
        """Retourner les skills pertinents formatés pour injection dans le prompt LLM"""
        query = ' '.join(filter(None, [
            task.get('title', ''),
            task.get('context', ''),
            task.get('intent', ''),
        ]))
        relevant = self.search(query, max_results=3)
        if not relevant:
            return ''

        lines = ['## Skills pertinents (patterns connus de Workflow)']
        for skill in relevant:
            lines.append(f"\n### {skill['name']}")
            if skill['description']:
                lines.append(skill['description'])
            # Limiter la taille pour ne pas saturer le contexte
            lines.append(skill['content'][:400])

        return '\n'.join(lines)

    def list_all(self) -> list[dict]:
        """Lister tous les skills avec leur métadonnées (pour le Curator)"""
        result = []
        for skill_file in self._all_skill_files():
            try:
                content = skill_file.read_text(encoding='utf-8')
                fm, _ = self._parse_frontmatter(content)
                result.append({
                    'path': str(skill_file),
                    'name': fm.get('name', skill_file.stem),
                    'description': fm.get('description', ''),
                    'tags': fm.get('tags', []),
                    'usage_count': fm.get('usage_count', 0),
                    'last_used': fm.get('last_used'),
                    'learned_at': fm.get('learned_at'),
                    'is_global': str(self.global_skills_dir) in str(skill_file),
                })
            except OSError:
                continue
        return result
```

## Intégration avec les autres modules

**Dans `ContextManager` (Phase 2)** — injecter les skills dans le prompt :
```python
# Niveau 3 — contexte tâche
skill_context = self.skill_manager.get_skills_for_context(task)
if skill_context:
    prompt = skill_context + '\n\n' + prompt
```

**Dans `ExecutionLoop` (Phase 3)** — créer un skill après succès avec retry :
```python
if len(error_history) >= 2 and success:
    await self._maybe_create_skill(task, error_history, code_response)
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `search('cannot find module')` retourne les skills pertinents | ⬜ |
| 2 | `create_skill(data, scope='local')` crée dans `.workflow/skills/` | ⬜ |
| 3 | `create_skill(data, scope='global')` crée dans `~/.workflow/skills/` | ⬜ |
| 4 | `get_skills_for_context(task)` retourne une string injectée dans le prompt | ⬜ |
| 5 | `record_usage()` incrémente `usage_count` dans le frontmatter YAML | ⬜ |
| 6 | `search()` cherche dans les skills globaux ET locaux | ⬜ |
| 7 | `list_all()` retourne tous les skills avec leurs métadonnées | ⬜ |
| 8 | `_parse_frontmatter()` gère les YAML malformés sans exception | ⬜ |
| 9 | Tests utilisent un répertoire temporaire `tmp_path` | ⬜ |
