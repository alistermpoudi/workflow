# Phase 1 — Tâche 1.1 : Initialisation du Projet

## Objectif

Créer la structure complète du projet avec les bons outils de développement Python : `pyproject.toml`, structure `src/workflow/`, configuration Ruff, Pytest, et les fichiers de config nécessaires.

## Fichiers à Créer

```
workflow/
├── pyproject.toml
├── .python-version          (contient "3.12")
├── .gitignore
├── bin/
│   └── .gitkeep             (workflow-mcp ajouté en Phase 4)
├── src/
│   └── workflow/
│       ├── __init__.py
│       ├── core/
│       │   └── __init__.py
│       ├── phases/
│       │   └── __init__.py
│       ├── tools/
│       │   └── __init__.py
│       ├── interfaces/
│       │   └── __init__.py
│       └── llm/
│           └── __init__.py
└── tests/
    └── unit/
        └── .gitkeep
```

## Implémentation

### `pyproject.toml`

```toml
[project]
name = "workflow"
version = "0.1.0"
description = "Agent de code avec mémoire projet persistante"
requires-python = ">=3.12"
dependencies = [
    "anthropic>=0.40.0",
    "litellm>=1.40.0",
    "mcp>=1.0.0",
    "aiosqlite>=0.20.0",
    "aiofiles>=24.0.0",
    "typer>=0.12.0",
    "rich>=13.7.0",
    "watchfiles>=0.24.0",
    "pyyaml>=6.0.0",
    "pydantic>=2.0.0",
    "fastembed>=0.3.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-mock>=3.12.0",
    "ruff>=0.4.0",
]

[project.scripts]
workflow = "workflow.interfaces.cli:app"
workflow-mcp = "workflow.interfaces.mcp_server:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/workflow"]

[tool.ruff]
line-length = 100
target-version = "py312"
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]
ignore = ["E501"]

[tool.ruff.lint.isort]
known-first-party = ["workflow"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
```

### `.python-version`

```
3.12
```

### `.gitignore`

```
.venv/
__pycache__/
*.pyc
*.pyo
.ruff_cache/
.pytest_cache/
dist/
*.egg-info/
.env
*.db
*.db-journal
*.db-wal
```

## Commandes de base

```bash
# Installer uv (si pas encore fait)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Créer l'environnement et installer les dépendances
uv sync --extra dev

# Lancer les tests
uv run pytest

# Linter
uv run ruff check src/ tests/

# Formatter
uv run ruff format src/ tests/

# Build + validate (équivalent npm run build:validate)
uv run ruff check src/ tests/ && uv run pytest
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `uv sync --extra dev` s'exécute sans erreur | ⬜ |
| 2 | `uv run ruff check src/ tests/` passe sans erreur | ⬜ |
| 3 | `uv run pytest` s'exécute (0 tests = OK pour cette tâche) | ⬜ |
| 4 | Structure de dossiers correcte avec `__init__.py` dans chaque module | ⬜ |
| 5 | `workflow --help` affiche l'aide (après Phase 3) | ⬜ |
