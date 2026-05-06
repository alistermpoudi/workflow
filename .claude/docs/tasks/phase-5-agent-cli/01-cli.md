# Phase 3 — Tâche 3.2 : CLI.py (typer + rich)

## Objectif

Créer l'interface CLI avec **typer** (commandes) et **rich** (affichage). Inclut la classe `RichIO` — l'objet `io` passé à toutes les phases. Chaque commande `workflow <cmd>` est déclarée ici et délègue à `WorkflowAgent`.

## Dépendances

- Phase 1 et 2 complètes ✅
- Tâche 3.3 (WorkflowAgent) pour le câblage final

## Fichiers à Créer

- `src/workflow/interfaces/cli.py` [CRÉER]

## Implémentation

```python
# src/workflow/interfaces/cli.py
import asyncio

import typer
from rich.console import Console
from rich.panel import Panel
from rich.text import Text
from rich.prompt import Prompt, Confirm

app = typer.Typer(
    name='workflow',
    help='Workflow — agent de code qui ne perd jamais le fil.',
    add_completion=False,
)

console = Console()


class RichIO:
    """
    Objet io injecté dans toutes les phases.
    Encapsule Console rich + typer.prompt pour les interactions.
    """

    def __init__(self, con: Console | None = None):
        self._console = con or Console()

    # ─── Affichage ────────────────────────────────────────────────────────────

    def print(self, message: str = '') -> None:
        self._console.print(message)

    def print_header(self, title: str) -> None:
        self._console.print(Panel(Text(title, style='bold white'), style='blue'))

    def print_section(self, title: str) -> None:
        self._console.print(f'\n[bold cyan]── {title} ──[/bold cyan]')

    def print_info(self, message: str) -> None:
        self._console.print(f'[dim]  {message}[/dim]')

    def print_success(self, message: str) -> None:
        self._console.print(f'[bold green]✓  {message}[/bold green]')

    def print_warning(self, message: str) -> None:
        self._console.print(f'[bold yellow]⚠  {message}[/bold yellow]')

    def print_error(self, message: str) -> None:
        self._console.print(f'[bold red]✗  {message}[/bold red]')

    # ─── Interaction ──────────────────────────────────────────────────────────

    def prompt(self, message: str, default: str = '', multiline: bool = False) -> str:
        if multiline:
            self._console.print(f'\n[bold]{message}[/bold] (terminer avec une ligne vide)\n')
            lines = []
            while True:
                try:
                    line = input()
                    if line == '' and lines:
                        break
                    lines.append(line)
                except EOFError:
                    break
            return '\n'.join(lines)

        return Prompt.ask(
            f'\n[bold]{message}[/bold]',
            default=default,
            console=self._console,
        )

    def confirm(self, message: str, default: bool = False) -> bool:
        return Confirm.ask(
            f'\n[bold]{message}[/bold]',
            default=default,
            console=self._console,
        )


# ─── Commandes CLI ────────────────────────────────────────────────────────────

@app.command()
def init(
    path: str = typer.Argument('.', help='Dossier du projet cible'),
) -> None:
    """Initialiser un nouveau projet Workflow."""
    from workflow.core.workflow_agent import WorkflowAgent
    agent = WorkflowAgent(path, io=RichIO())
    asyncio.run(agent.init())


@app.command()
def run(
    path: str = typer.Argument('.', help='Dossier du projet cible'),
) -> None:
    """Reprendre ou avancer le projet (phase courante)."""
    from workflow.core.workflow_agent import WorkflowAgent
    agent = WorkflowAgent(path, io=RichIO())
    asyncio.run(agent.run_phase())


@app.command()
def status(
    path: str = typer.Argument('.', help='Dossier du projet cible'),
) -> None:
    """Afficher l'état courant du projet."""
    from workflow.core.workflow_agent import WorkflowAgent
    agent = WorkflowAgent(path, io=RichIO())
    asyncio.run(agent.show_status())


@app.command()
def task(
    task_id: str = typer.Argument(..., help='ID de la tâche (ex: TASK-003)'),
    path: str = typer.Option('.', '--path', '-p', help='Dossier du projet cible'),
) -> None:
    """Exécuter une tâche spécifique."""
    from workflow.core.workflow_agent import WorkflowAgent
    agent = WorkflowAgent(path, io=RichIO())
    asyncio.run(agent.run_task(task_id))


@app.command()
def onboard(
    path: str = typer.Argument('.', help='Dossier du projet cible'),
) -> None:
    """Onboarding instantané — résumé complet du projet pour un nouveau développeur."""
    from workflow.core.workflow_agent import WorkflowAgent
    agent = WorkflowAgent(path, io=RichIO())
    asyncio.run(agent.onboard())


@app.command()
def daemon(
    action: str = typer.Argument('start', help='start | stop | status'),
    path: str = typer.Option('.', '--path', '-p', help='Dossier du projet cible'),
) -> None:
    """Gérer le daemon Heartbeat (surveillance en arrière-plan)."""
    from workflow.core.daemon_heartbeat import DaemonHeartbeat
    dh = DaemonHeartbeat(path, io=RichIO())
    if action == 'start':
        dh.start()
    elif action == 'stop':
        dh.stop()
    elif action == 'status':
        dh.show_status()
    else:
        typer.echo(f'Action inconnue : {action}. Utilise start, stop ou status.')
        raise typer.Exit(1)


def main() -> None:
    app()
```

## Point d'Entrée dans `pyproject.toml`

```toml
[project.scripts]
workflow = "workflow.interfaces.cli:main"
workflow-mcp = "workflow.interfaces.mcp_server:main"
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `workflow init .` initialise le projet via `WorkflowAgent.init()` | ⬜ |
| 2 | `workflow run .` reprend la phase courante | ⬜ |
| 3 | `workflow status` affiche la phase, la version active, et les tâches pending/done | ⬜ |
| 4 | `workflow task TASK-003` exécute une tâche spécifique | ⬜ |
| 5 | `RichIO.print_header()` affiche un panel rich bleu | ⬜ |
| 6 | `RichIO.print_success()` affiche en vert avec ✓ | ⬜ |
| 7 | `RichIO.prompt(multiline=True)` collecte plusieurs lignes jusqu'à une ligne vide | ⬜ |
| 8 | `RichIO.prompt()` avec `default` affiche la valeur par défaut | ⬜ |
| 9 | `workflow daemon start` lance le daemon sans bloquer le terminal | ⬜ |
| 10 | `workflow onboard` génère le résumé d'onboarding via `WorkflowAgent.onboard()` | ⬜ |
