# Phase 1 — Tâche 1.5 : DecisionsLog.py (SQLite)

## Objectif

Créer le module `DecisionsLog.py` — le journal actif des décisions techniques, stocké dans **SQLite avec FTS5**. Remplace le fichier texte `decisions.log` + `decisions-graph.json` de la spec initiale. SQLite offre la recherche full-text, la concurrence WAL, et la détection de contradictions via requêtes SQL.

## Dépendances

- Tâche 1.2 ✅ (`FileSystem.py`)

## Fichiers à Créer / Modifier

- `src/workflow/core/decisions_log.py` [CRÉER]
- `tests/unit/test_decisions_log.py` [CRÉER]

## Schéma SQLite

```sql
CREATE TABLE IF NOT EXISTS decisions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    decision_id TEXT UNIQUE,
    task_id TEXT NOT NULL,
    summary TEXT NOT NULL,
    reason TEXT NOT NULL,
    date TEXT NOT NULL,
    confidence TEXT DEFAULT 'HIGH',
    scope TEXT DEFAULT 'global',
    revisable INTEGER DEFAULT 1,
    created_at TEXT NOT NULL
);

CREATE VIRTUAL TABLE IF NOT EXISTS decisions_fts USING fts5(
    summary, reason, task_id,
    content='decisions',
    content_rowid='id'
);

CREATE TABLE IF NOT EXISTS decision_relations (
    source_id TEXT,
    target_id TEXT,
    relation_type TEXT,
    reason TEXT,
    PRIMARY KEY (source_id, target_id, relation_type)
);
```

**Types de relations** : `DEPENDS_ON` | `CONTRADICTS` | `SUPERSEDES` | `REFINES`

**Niveaux de confiance** : `HIGH` (décision actée) | `MEDIUM` (à revalider) | `LOW` (expérimentale)

## Implémentation

```python
# src/workflow/core/decisions_log.py
import aiosqlite
from datetime import date as date_cls
from pathlib import Path


SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS decisions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    decision_id TEXT UNIQUE,
    task_id TEXT NOT NULL,
    summary TEXT NOT NULL,
    reason TEXT NOT NULL,
    date TEXT NOT NULL,
    confidence TEXT DEFAULT 'HIGH',
    scope TEXT DEFAULT 'global',
    revisable INTEGER DEFAULT 1,
    created_at TEXT NOT NULL
);

CREATE VIRTUAL TABLE IF NOT EXISTS decisions_fts USING fts5(
    summary, reason, task_id,
    content='decisions',
    content_rowid='id'
);

CREATE TABLE IF NOT EXISTS decision_relations (
    source_id TEXT,
    target_id TEXT,
    relation_type TEXT,
    reason TEXT,
    PRIMARY KEY (source_id, target_id, relation_type)
);
"""


class DecisionsLog:
    def __init__(self, project_root: str):
        self.db_path = Path(project_root) / '.workflow' / 'decisions.db'

    async def _connect(self) -> aiosqlite.Connection:
        """Ouvrir la connexion SQLite avec WAL et initialiser le schéma"""
        self.db_path.parent.mkdir(parents=True, exist_ok=True)
        db = await aiosqlite.connect(self.db_path)
        db.row_factory = aiosqlite.Row
        await db.execute('PRAGMA journal_mode=WAL')
        await db.executescript(SCHEMA_SQL)
        await db.commit()
        return db

    async def log(
        self,
        task_id: str,
        summary: str,
        reason: str,
        options: dict | None = None,
    ) -> dict:
        """Enregistrer une nouvelle décision dans SQLite + FTS"""
        options = options or {}
        today = date_cls.today().isoformat()

        async with await self._connect() as db:
            # Générer l'ID séquentiel
            async with db.execute('SELECT COUNT(*) FROM decisions') as cur:
                row = await cur.fetchone()
                decision_id = f"DECISION-{row[0] + 1:03d}"

            await db.execute(
                """INSERT INTO decisions
                   (decision_id, task_id, summary, reason, date,
                    confidence, scope, revisable, created_at)
                   VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)""",
                (
                    decision_id, task_id, summary, reason, today,
                    options.get('confidence', 'HIGH'),
                    options.get('scope', 'global'),
                    1 if options.get('revisable', True) else 0,
                    today,
                ),
            )

            # Mettre à jour le FTS
            await db.execute(
                """INSERT INTO decisions_fts(rowid, summary, reason, task_id)
                   SELECT id, summary, reason, task_id
                   FROM decisions WHERE decision_id = ?""",
                (decision_id,),
            )

            # Enregistrer les relations
            for rel in options.get('relations', []):
                await db.execute(
                    'INSERT OR IGNORE INTO decision_relations VALUES (?, ?, ?, ?)',
                    (decision_id, rel['target'], rel['type'], rel.get('reason', '')),
                )

            await db.commit()

        contradictions = await self.check_contradictions(decision_id)
        result: dict = {'id': decision_id}
        if contradictions:
            result['contradictions'] = contradictions
        return result

    async def get_recent(self, n: int = 5) -> list[dict]:
        """Retourner les N dernières décisions"""
        async with await self._connect() as db:
            async with db.execute(
                'SELECT * FROM decisions ORDER BY id DESC LIMIT ?', (n,)
            ) as cur:
                rows = await cur.fetchall()
        return [dict(row) for row in rows]

    async def get_relevant(self, task: dict) -> list[dict]:
        """Recherche FTS5 par mots-clés extraits de la tâche"""
        keywords = self._extract_keywords(task)
        if not keywords:
            return []

        # FTS5 MATCH avec OR entre les mots-clés
        fts_query = ' OR '.join(f'"{kw}"' for kw in keywords)

        async with await self._connect() as db:
            try:
                async with db.execute(
                    """SELECT d.* FROM decisions d
                       JOIN decisions_fts ON d.id = decisions_fts.rowid
                       WHERE decisions_fts MATCH ?
                       ORDER BY rank LIMIT 10""",
                    (fts_query,),
                ) as cur:
                    rows = await cur.fetchall()
            except Exception:
                # FTS peut échouer sur des termes mal formés — fallback silencieux
                rows = []
        return [dict(row) for row in rows]

    async def get_all(self) -> list[dict]:
        """Retourner toutes les décisions avec confidence et scope"""
        async with await self._connect() as db:
            async with db.execute('SELECT * FROM decisions ORDER BY id') as cur:
                rows = await cur.fetchall()
        return [dict(row) for row in rows]

    async def log_many(self, task_id: str, decisions: list[dict]):
        """Enregistrer plusieurs décisions d'une traite"""
        for d in decisions:
            await self.log(task_id, d['decision'], d['reason'])

    async def check_contradictions(self, new_id: str | None = None) -> list[dict]:
        """Détecter les contradictions actives dans les relations"""
        async with await self._connect() as db:
            base_query = """
                SELECT r.source_id, r.target_id, r.reason
                FROM decision_relations r
                WHERE r.relation_type = 'CONTRADICTS'
                AND NOT EXISTS (
                    SELECT 1 FROM decision_relations s
                    WHERE s.relation_type = 'SUPERSEDES'
                    AND (s.target_id = r.source_id OR s.target_id = r.target_id)
                )
            """
            if new_id:
                async with db.execute(
                    base_query + ' AND r.source_id = ?', (new_id,)
                ) as cur:
                    rows = await cur.fetchall()
            else:
                async with db.execute(base_query) as cur:
                    rows = await cur.fetchall()

        return [
            {'source': row['source_id'], 'target': row['target_id'], 'reason': row['reason']}
            for row in rows
        ]

    def _extract_keywords(self, task: dict) -> list[str]:
        """Extraire des mots-clés pertinents d'une tâche"""
        technical_terms = [
            'ORM', 'auth', 'JWT', 'session', 'database', 'DB', 'SQL',
            'API', 'REST', 'GraphQL', 'cache', 'Redis', 'migration',
            'test', 'lint', 'build', 'deploy', 'CI', 'Docker',
            'async', 'prisma', 'sqlalchemy', 'pydantic',
        ]
        keywords: list[str] = []
        title_words = (task.get('title') or '').split()
        keywords.extend(w for w in title_words if len(w) > 3)

        context = (task.get('context') or '') + ' ' + (task.get('title') or '')
        keywords.extend(t for t in technical_terms if t.lower() in context.lower())
        return list(set(keywords))
```

## Utilisation dans `ExecutionPhase` (preview Phase 2)

```python
# Avant de coder une tâche — consulter les décisions pertinentes
relevant = await decisions_log.get_relevant(task)
if relevant:
    console.print('[dim]Décisions antérieures pertinentes :[/dim]')
    for d in relevant:
        console.print(f"  • {d['summary']}")
# → Ces décisions sont injectées dans le prompt LLM
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `log()` insère dans SQLite ET met à jour le FTS | ⬜ |
| 2 | `get_recent(5)` retourne les 5 dernières entrées | ⬜ |
| 3 | `get_all()` retourne toutes les entrées avec `confidence` et `scope` | ⬜ |
| 4 | `get_relevant(task)` effectue une vraie recherche FTS5 | ⬜ |
| 5 | `check_contradictions()` détecte un conflit `CONTRADICTS` actif | ⬜ |
| 6 | `check_contradictions()` ignore une contradiction si `SUPERSEDES` existe | ⬜ |
| 7 | `log()` retourne `{ id, contradictions }` si contradiction détectée | ⬜ |
| 8 | Retourne `[]` (pas d'exception) si `decisions.db` n'existe pas encore | ⬜ |
| 9 | Tests utilisent un répertoire temporaire `tmp_path` (fixture pytest) | ⬜ |
| 10 | `log_many()` enregistre plusieurs décisions séquentiellement | ⬜ |
