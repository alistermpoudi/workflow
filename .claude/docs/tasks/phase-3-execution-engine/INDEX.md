# Phase 3 — Boucle d'Exécution

## Objectif

Construire le moteur d'exécution avec **boucle skill → curator → réutilisation** (Pilier 2 complet). `ExecutionLoop` génère des patches via `CodePatcher`, valide, se corrige (max 3 retries), et **crée automatiquement un skill après retry réussi**. Le `SkillCurator` consolide périodiquement pour éviter le bruit.

C'est ici que Workflow devient *un agent qui apprend*.

## Piliers Adressés

- **Pilier 2 (complet)** — Skill → Curator → Réutilisation (tâches 3.1 + 3.2)
- **Pilier 5 (utilisé)** — `ExecutionLoop` génère et applique via `CodePatcher`
- **Pilier 3 (utilisé)** — Routage par rôle (`code_generation`, `curator`, `fast`)

## Stack Phase 3

```
asyncio              Boucle async native
subprocess           build_validate + test (via allowed_commands)
litellm              Génération de code (role='code_generation' + 'curator')
SkillManager         Phase 1 (CRUD + recherche)
CodePatcher          Phase 2 (application chirurgicale)
```

## Tâches

| Tâche | Fichier | Description |
|-------|---------|-------------|
| 3.1 | [01-execution-loop.md](01-execution-loop.md) | `ExecutionLoop.py` — génère patches, applique, valide, retry (max 3), crée skill auto |
| 3.2 | [02-skill-curator.md](02-skill-curator.md) | **`SkillCurator.py`** — consolidation LLM périodique des skills (dédup, promotion cross-context, archivage) |
| 3.3 | [03-execution-extensions.md](03-execution-extensions.md) | Extensions ExecutionLoop : TDD, Security Review, Architecture Review, Dependency Intelligence |
| 3.4 | [04-teach-system.md](04-teach-system.md) | **`TeachSystem.py`** — `workflow teach` / `workflow avoid` + niveau `USER_OVERRIDE` (Pilier 2 — source `user_explicit`) |

## Dépendances

- Phase 1 ✅
- Phase 2 ✅

## Critères de Sortie de Phase

- [ ] `ExecutionLoop` génère et applique des patches (jamais des fichiers complets) via `CodePatcher`
- [ ] `ExecutionLoop` valide avec `build_validate` + tests, se corrige jusqu'à 3 fois
- [ ] Après un retry réussi, `ExecutionLoop` crée un skill dans `~/.workflow/skills/`
- [ ] `ExecutionLoop` consulte `SkillManager.find_fix_for_error()` AVANT chaque retry
- [ ] `SkillCurator.run()` détecte les doublons exacts (fingerprint) sans LLM
- [ ] `SkillCurator._llm_consolidate()` batche par 50 skills si > 50
- [ ] `SkillCurator._select_for_promotion()` ne promeut qu'avec usage ≥ 3 ET ≥ 2 projets
- [ ] `SkillCurator._select_for_archive()` archive seulement les skills 90j+ avec usage = 0
- [ ] `dry_run=True` ne modifie aucun fichier, retourne le plan
- [ ] Mode TDD : génère les tests avant le code
- [ ] Mode Security Review bloque sur les failles critiques détectées
- [ ] Mode Architecture Review force un retry sur violation architecturale
- [ ] `workflow teach <rule>` crée un skill `USER_OVERRIDE` dans le context actif
- [ ] `workflow avoid <rule>` ajoute un anti-pattern dans `avoid.md` du context
- [ ] `--personal` place le skill/avoid dans `<context>/personal/` (non partagé via git)
- [ ] `SkillCurator` n'archive jamais un skill `USER_OVERRIDE` automatiquement
- [ ] `SkillCurator` peut promouvoir un pattern présent dans 2+ contexts vers leur ancêtre commun
