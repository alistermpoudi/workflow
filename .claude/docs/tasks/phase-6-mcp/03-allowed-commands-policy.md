# Phase 6 — Tâche 6.3 : AllowedCommandsPolicy.py

## Objectif

Créer `AllowedCommandsPolicy.py` — la couche de **sécurité avec apprentissage** pour les commandes shell exécutables par Workflow. Whitelist initiale + prompt utilisateur "approuver une fois et mémoriser" pour les nouvelles commandes. Politique partageable via `.workflow/allowed-commands.json`.

> **Reframe du design initial** : la whitelist gelée dans `tech-stack.json` tue le flow dev (un dev tape des nouvelles commandes en permanence : `pip install X`, `docker logs`, `git rebase`...). On veut sécurité **et** flexibilité.

## Dépendances

- Tâche 6.1 ✅ (`MCPServer`)
- Phase 1 ✅ (`FileSystem`, `ProjectMemory`)

## Fichiers à Créer

- `src/workflow/interfaces/allowed_commands_policy.py` [CRÉER]
- `tests/unit/test_allowed_commands_policy.py` [CRÉER]

## Politique en 3 Niveaux

### Niveau 1 — Auto-allow (built-in)

Commandes universellement sûres, pré-approuvées :

```python
BUILTIN_SAFE_COMMANDS = [
    # Lecture / inspection
    'git status', 'git log', 'git diff', 'git branch', 'git show',
    'ls', 'pwd', 'cat', 'head', 'tail', 'wc', 'grep', 'find', 'tree',
    'echo', 'printf',

    # Build/test (toolchain standard)
    'uv run pytest', 'uv run ruff', 'uv run mypy',
    'npm test', 'npm run lint', 'npm run build',
    'cargo test', 'cargo check', 'cargo clippy',
    'pytest', 'mypy', 'ruff',
]
```

Exécutables sans demander.

### Niveau 2 — Project allowed (`.workflow/allowed-commands.json`)

Commandes approuvées au niveau du projet, partageables via git :

```json
{
  "schema_version": "1.0.0",
  "allowed": [
    {
      "command": "uv run alembic",
      "approved_at": "2026-04-12T14:30:00Z",
      "approved_by": "alister",
      "scope": "prefix"
    },
    {
      "command": "docker compose up",
      "approved_at": "2026-04-15T09:00:00Z",
      "approved_by": "alister",
      "scope": "exact"
    }
  ],
  "denied": [
    {
      "command": "rm -rf",
      "denied_at": "2026-04-12T14:31:00Z",
      "reason": "trop dangereux"
    }
  ]
}
```

`scope = "prefix"` autorise toute commande commençant par cette chaîne (ex : `uv run alembic upgrade head`).
`scope = "exact"` exige correspondance exacte.

### Niveau 3 — Prompt à l'utilisateur

Pour toute commande hors Niveau 1/2 :

```
Workflow : "Je veux exécuter : prisma migrate dev --name add-users"
           Approuver ?
           [O] Oui, une fois
           [P] Oui et mémoriser (préfixe : 'prisma migrate')
           [E] Oui et mémoriser (exact)
           [N] Non
           [B] Non et bannir
```

Si en mode non-interactif (CI, daemon), bloque par défaut et logue la commande à valider plus tard.

## Implémentation

```python
# src/workflow/interfaces/allowed_commands_policy.py
import json
from pathlib import Path
from datetime import datetime, timezone
from typing import Literal
from dataclasses import dataclass, asdict

from workflow.tools.filesystem import FileSystem

PolicyDecision = Literal['ALLOW', 'DENY', 'ASK']
ApprovalScope = Literal['exact', 'prefix']

POLICY_FILE = 'allowed-commands.json'

BUILTIN_SAFE_COMMANDS = [
    'git status', 'git log', 'git diff', 'git branch', 'git show',
    'ls', 'pwd', 'cat', 'head', 'tail', 'wc', 'grep', 'find', 'tree',
    'echo', 'printf',
    'uv run pytest', 'uv run ruff', 'uv run mypy',
    'npm test', 'npm run lint', 'npm run build',
    'cargo test', 'cargo check', 'cargo clippy',
    'pytest', 'mypy', 'ruff',
]

ALWAYS_DENIED_PATTERNS = [
    'rm -rf /',
    'sudo rm',
    'chmod 777',
    'curl | sh',
    'wget | sh',
    '> /dev/sda',
    'dd if=',
]


@dataclass
class PolicyCheck:
    decision: PolicyDecision
    command: str
    rule: str  # 'builtin', 'project_allow', 'project_deny', 'always_deny', 'unknown'
    matched_entry: dict | None = None


class AllowedCommandsPolicy:
    def __init__(self, project_root: str, io=None, interactive: bool = True):
        self.project_root = Path(project_root)
        self.fs = FileSystem(project_root)
        self.io = io
        self.interactive = interactive
        self._policy: dict | None = None

    async def _load(self) -> dict:
        if self._policy is not None:
            return self._policy
        data = await self.fs.read_json(POLICY_FILE)
        self._policy = data or {
            'schema_version': '1.0.0',
            'allowed': [],
            'denied': [],
        }
        return self._policy

    async def _save(self):
        await self.fs.write_json_atomic(POLICY_FILE, self._policy)

    # ─── Vérification ─────────────────────────────────────────────────

    async def check(self, command: str) -> PolicyCheck:
        """
        Vérifier si une commande est autorisée selon la politique.
        Ordre : ALWAYS_DENIED → builtin → project_deny → project_allow → unknown.
        """
        cmd_stripped = command.strip()

        # Always denied (sécurité absolue)
        for pattern in ALWAYS_DENIED_PATTERNS:
            if pattern in cmd_stripped:
                return PolicyCheck(
                    decision='DENY',
                    command=cmd_stripped,
                    rule='always_deny',
                )

        # Builtin safe
        for safe in BUILTIN_SAFE_COMMANDS:
            if cmd_stripped == safe or cmd_stripped.startswith(safe + ' '):
                return PolicyCheck(
                    decision='ALLOW',
                    command=cmd_stripped,
                    rule='builtin',
                )

        policy = await self._load()

        # Project denied
        for entry in policy['denied']:
            if self._matches(cmd_stripped, entry['command'], entry.get('scope', 'exact')):
                return PolicyCheck(
                    decision='DENY',
                    command=cmd_stripped,
                    rule='project_deny',
                    matched_entry=entry,
                )

        # Project allowed
        for entry in policy['allowed']:
            if self._matches(cmd_stripped, entry['command'], entry.get('scope', 'exact')):
                return PolicyCheck(
                    decision='ALLOW',
                    command=cmd_stripped,
                    rule='project_allow',
                    matched_entry=entry,
                )

        # Unknown — demander à l'utilisateur si interactif, sinon DENY
        return PolicyCheck(
            decision='ASK' if self.interactive else 'DENY',
            command=cmd_stripped,
            rule='unknown',
        )

    @staticmethod
    def _matches(command: str, pattern: str, scope: ApprovalScope) -> bool:
        if scope == 'exact':
            return command == pattern
        if scope == 'prefix':
            return command == pattern or command.startswith(pattern + ' ')
        return False

    # ─── Apprentissage interactif ─────────────────────────────────────

    async def ask_user(self, command: str) -> PolicyDecision:
        """Demander à l'utilisateur quoi faire d'une commande inconnue"""
        if not self.interactive or self.io is None:
            return 'DENY'

        self.io.print_warning(f'Commande non autorisée : {command!r}')
        choice = self.io.prompt(
            'Approuver ? [O]ui une fois / [P]réfixe / [E]xact / [N]on / [B]annir',
            default='N',
        ).strip().lower()

        if choice in ('o', 'oui', 'y', 'yes'):
            return 'ALLOW'  # Une fois, pas de persistance
        if choice == 'p':
            await self._add_allowed(command, scope='prefix')
            return 'ALLOW'
        if choice == 'e':
            await self._add_allowed(command, scope='exact')
            return 'ALLOW'
        if choice == 'b':
            await self._add_denied(command, reason='bani par utilisateur')
            return 'DENY'
        return 'DENY'

    async def _add_allowed(self, command: str, scope: ApprovalScope):
        await self._load()
        # Si scope=prefix, extraire le préfixe (les 2-3 premiers mots typiquement)
        if scope == 'prefix':
            tokens = command.split()
            command = ' '.join(tokens[:min(3, len(tokens))])

        # Éviter doublons
        if any(e['command'] == command for e in self._policy['allowed']):
            return

        self._policy['allowed'].append({
            'command': command,
            'approved_at': datetime.now(timezone.utc).isoformat(),
            'approved_by': self._current_user(),
            'scope': scope,
        })
        await self._save()
        if self.io:
            self.io.print_info(f'Mémorisé : {command} ({scope})')

    async def _add_denied(self, command: str, reason: str):
        await self._load()
        self._policy['denied'].append({
            'command': command,
            'denied_at': datetime.now(timezone.utc).isoformat(),
            'reason': reason,
        })
        await self._save()

    @staticmethod
    def _current_user() -> str:
        import os
        return os.environ.get('USER', 'unknown')

    # ─── API pour MCPServer ───────────────────────────────────────────

    async def authorize(self, command: str) -> bool:
        """
        Point d'entrée unique pour MCPServer.
        Retourne True si autorisé, False sinon. Demande à l'utilisateur si nécessaire.
        """
        check = await self.check(command)
        if check.decision == 'ALLOW':
            return True
        if check.decision == 'DENY':
            if self.io:
                self.io.print_error(
                    f'Commande refusée ({check.rule}) : {command}'
                )
            return False
        # ASK
        decision = await self.ask_user(command)
        return decision == 'ALLOW'
```

## Intégration avec `MCPServer` (Phase 6.1)

```python
# Dans MCPServer.create_server()
policy = AllowedCommandsPolicy(project_root, io=io, interactive=True)

@server.call_tool()
async def workflow_run_command(args: dict) -> list:
    command = args['command']
    if not await policy.authorize(command):
        raise PermissionError(f'Commande non autorisée : {command}')
    ...
```

## Bootstrap depuis `tech-stack.json`

À l'init du projet, `ArchitecturePhase` (Phase 4) génère un `allowed-commands.json` initial à partir des `allowed_commands` proposées par le LLM :

```python
# Dans ArchitecturePhase.run()
proposed = stack.get('allowed_commands', [])
policy = AllowedCommandsPolicy(project_root, interactive=False)
for cmd in proposed:
    await policy._add_allowed(cmd, scope='prefix')
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `check('git status')` retourne `ALLOW`, rule='builtin' | ⬜ |
| 2 | `check('rm -rf /')` retourne `DENY`, rule='always_deny' | ⬜ |
| 3 | `check('uv run alembic upgrade')` retourne `ALLOW` après ajout en prefix | ⬜ |
| 4 | `check('inconnue')` retourne `ASK` en mode interactif, `DENY` sinon | ⬜ |
| 5 | `_add_allowed(cmd, 'prefix')` extrait les 3 premiers tokens | ⬜ |
| 6 | `_add_allowed()` évite les doublons (même commande non ajoutée 2x) | ⬜ |
| 7 | Le fichier `allowed-commands.json` valide le schéma JSON | ⬜ |
| 8 | `authorize()` consulte la politique ET demande à l'utilisateur si nécessaire | ⬜ |
| 9 | En mode `interactive=False`, les commandes inconnues sont DENY (pas hang) | ⬜ |
| 10 | Les commandes dans `denied` ne peuvent pas être ré-allowées sans suppression manuelle | ⬜ |

## Notes d'Implémentation

- **Sécurité d'abord** : `ALWAYS_DENIED_PATTERNS` est une liste hardcodée non-overridable. Pas d'option pour la désactiver.
- **CI-friendly** : en CI/CD, lancer Workflow avec `interactive=False` — les commandes inconnues échouent au lieu de hang. La whitelist projet doit être à jour avant le run.
- **Audit trail** : chaque approbation est tracée avec `approved_by` et `approved_at`. Permet à un dev junior de voir ce qu'un senior a autorisé sur le projet.
- **Partage via git** : `allowed-commands.json` est commité comme tout autre artéfact `.workflow/`. Onboarding instantané d'un nouveau dev.
