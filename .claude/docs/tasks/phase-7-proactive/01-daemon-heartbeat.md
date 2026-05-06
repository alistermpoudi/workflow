# Phase 3 — Tâche 3.4 : DaemonHeartbeat.py

## Objectif

Créer `DaemonHeartbeat.py` — le daemon proactif qui tourne en arrière-plan. Il envoie un briefing matin, surveille les builds CI, et propose la prochaine tâche automatiquement. **Ne bloque jamais l'utilisateur.**

Le daemon est lancé via `subprocess.Popen(start_new_session=True)` et persiste dans un fichier PID dans `.workflow/daemon.pid`.

## Dépendances

- Phase 1 complète ✅

## Fichiers à Créer

- `src/workflow/core/daemon_heartbeat.py` [CRÉER]

## Implémentation

```python
# src/workflow/core/daemon_heartbeat.py
import asyncio
import json
import os
import sys
from datetime import datetime, timezone
from pathlib import Path
import subprocess

from workflow.core.project_memory import ProjectMemory
from workflow.llm.llm_provider import LLMProvider


PID_FILE = '.workflow/daemon.pid'
LOG_FILE = '.workflow/daemon.log'
BRIEFING_FILE = '.workflow/briefing.md'
HEARTBEAT_INTERVAL = 300  # 5 minutes


class DaemonHeartbeat:
    def __init__(self, project_root: str, io=None):
        self.project_root = Path(project_root).resolve()
        self.io = io
        self._pid_path = self.project_root / PID_FILE
        self._log_path = self.project_root / LOG_FILE
        self._briefing_path = self.project_root / BRIEFING_FILE

    # ─── Contrôle du daemon (appelé depuis la CLI) ────────────────────────────

    def start(self) -> None:
        if self._is_running():
            if self.io:
                self.io.print_warning(f'Daemon déjà en cours (PID {self._read_pid()}).')
            return

        # Lancer le daemon en arrière-plan (détaché du terminal)
        proc = subprocess.Popen(
            [sys.executable, '-m', 'workflow.core.daemon_heartbeat', '--daemon', str(self.project_root)],
            stdout=open(self._log_path, 'a'),
            stderr=subprocess.STDOUT,
            start_new_session=True,
        )
        self._write_pid(proc.pid)
        if self.io:
            self.io.print_success(f'Daemon démarré (PID {proc.pid}).')

    def stop(self) -> None:
        pid = self._read_pid()
        if not pid:
            if self.io:
                self.io.print_warning('Aucun daemon en cours.')
            return
        try:
            os.kill(pid, 15)  # SIGTERM
            self._pid_path.unlink(missing_ok=True)
            if self.io:
                self.io.print_success(f'Daemon arrêté (PID {pid}).')
        except ProcessLookupError:
            self._pid_path.unlink(missing_ok=True)
            if self.io:
                self.io.print_info('Daemon déjà arrêté (fichier PID nettoyé).')

    def show_status(self) -> None:
        if not self.io:
            return
        if self._is_running():
            self.io.print_success(f'Daemon actif — PID {self._read_pid()}')
        else:
            self.io.print_info('Daemon inactif.')

    # ─── Boucle principale du daemon ─────────────────────────────────────────

    async def run_daemon_loop(self) -> None:
        """Boucle principale — tourne indéfiniment en arrière-plan"""
        llm = LLMProvider.from_config_file()
        memory = ProjectMemory(str(self.project_root))

        self._log('Daemon démarré.')

        while True:
            try:
                now = datetime.now(timezone.utc)

                # Briefing du matin (entre 7h et 9h)
                if 7 <= now.hour < 9:
                    await self._generate_briefing(memory, llm)

                # Surveiller si des tâches sont bloquées (failed)
                await self._check_failed_tasks(memory)

            except Exception as e:
                self._log(f'Erreur heartbeat : {e}')

            await asyncio.sleep(HEARTBEAT_INTERVAL)

    async def _generate_briefing(self, memory: ProjectMemory, llm: LLMProvider) -> None:
        """Générer le briefing quotidien et l'écrire dans briefing.md"""
        # Éviter de régénérer si déjà fait aujourd'hui
        if self._briefing_path.exists():
            mtime = datetime.fromtimestamp(self._briefing_path.stat().st_mtime, tz=timezone.utc)
            if mtime.date() == datetime.now(timezone.utc).date():
                return

        project = await memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            return

        progress = await memory.get_progress(version)
        pending = (progress or {}).get('pending', [])
        done = (progress or {}).get('done', [])

        briefing_prompt = f"""Génère un briefing matinal concis pour le développeur.
Projet : {(project or {}).get('name', '?')}
Version active : {version}
Tâches terminées : {len(done)}
Tâches en attente : {', '.join(pending[:5]) or 'aucune'}

Inclus :
1. Résumé de l'avancement (1 phrase)
2. Prochaine tâche recommandée
3. Alertes éventuelles (tâches failed, etc.)

Format Markdown court."""

        briefing = await llm.ask(briefing_prompt, role='fast')
        timestamp = datetime.now(timezone.utc).strftime('%Y-%m-%d %H:%M UTC')
        content = f'# Briefing — {timestamp}\n\n{briefing}\n'
        self._briefing_path.write_text(content, encoding='utf-8')
        self._log(f'Briefing généré : {self._briefing_path}')

    async def _check_failed_tasks(self, memory: ProjectMemory) -> None:
        """Vérifier les tâches failed et logguer une alerte"""
        project = await memory.get_project()
        version = (project or {}).get('active_version')
        if not version:
            return

        progress = await memory.get_progress(version)
        failed = (progress or {}).get('failed', [])
        if failed:
            self._log(f'ALERTE : {len(failed)} tâche(s) échouée(s) — {", ".join(failed)}')

    # ─── Helpers PID ─────────────────────────────────────────────────────────

    def _is_running(self) -> bool:
        pid = self._read_pid()
        if not pid:
            return False
        try:
            os.kill(pid, 0)  # Signal 0 = vérifier existence
            return True
        except ProcessLookupError:
            return False

    def _read_pid(self) -> int | None:
        try:
            return int(self._pid_path.read_text().strip())
        except (FileNotFoundError, ValueError):
            return None

    def _write_pid(self, pid: int) -> None:
        self._pid_path.parent.mkdir(parents=True, exist_ok=True)
        self._pid_path.write_text(str(pid))

    def _log(self, message: str) -> None:
        timestamp = datetime.now(timezone.utc).strftime('%Y-%m-%d %H:%M:%S UTC')
        self._log_path.parent.mkdir(parents=True, exist_ok=True)
        with open(self._log_path, 'a') as f:
            f.write(f'[{timestamp}] {message}\n')


# ─── Point d'entrée daemon ────────────────────────────────────────────────────

if __name__ == '__main__':
    import sys
    if '--daemon' in sys.argv:
        project_root = sys.argv[sys.argv.index('--daemon') + 1]
        dh = DaemonHeartbeat(project_root)
        asyncio.run(dh.run_daemon_loop())
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `start()` lance le daemon avec `subprocess.Popen(start_new_session=True)` | ⬜ |
| 2 | `start()` écrit le PID dans `.workflow/daemon.pid` | ⬜ |
| 3 | `stop()` envoie SIGTERM et supprime le fichier PID | ⬜ |
| 4 | `_is_running()` utilise `os.kill(pid, 0)` pour vérifier l'existence du process | ⬜ |
| 5 | `_generate_briefing()` ne régénère pas si déjà fait aujourd'hui | ⬜ |
| 6 | Le briefing est écrit dans `.workflow/briefing.md` | ⬜ |
| 7 | `_generate_briefing()` utilise `role='fast'` | ⬜ |
| 8 | La boucle tourne toutes les `HEARTBEAT_INTERVAL` secondes sans bloquer | ⬜ |
| 9 | Les erreurs dans la boucle sont loggées sans faire crasher le daemon | ⬜ |
| 10 | `show_status()` indique si le daemon est actif et son PID | ⬜ |
