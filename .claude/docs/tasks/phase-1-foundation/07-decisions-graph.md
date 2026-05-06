# Phase 1 — Tâche 1.7 : DecisionsGraph

## Objectif

Créer `DecisionsGraph.py` — le **graphe actif des décisions techniques** avec relations typées, niveaux de confiance, détection automatique de contradictions, et **rétro-propagation** vers les tâches en cours impactées.

> **Pilier load-bearing #4.** C'est ce qui transforme Workflow d'un "exécutant qui suit des décisions" en "PM technique qui les fait respecter cohéremment dans le temps". Aucun outil actuel ne fait ça.

## Pourquoi un graphe, pas juste une liste

Une liste de décisions (`decisions.log`) répond à : "qu'a-t-on décidé ?"
Un graphe répond à :
- "cette nouvelle décision **contredit-elle** une ancienne ?"
- "si je change la décision X, **quelles tâches** sont impactées ?"
- "cette décision **dépend-elle** d'une autre qui pourrait changer ?"
- "y a-t-il des **clusters de décisions liées** sur l'auth, l'ORM, le cache… ?"

## Dépendances

- Tâche 1.6 ✅ (`DecisionsLog`)
- Tâche 1.4 ✅ (`TaskManager`)

## Fichiers à Créer

- `src/workflow/core/decisions_graph.py` [CRÉER]
- `tests/unit/test_decisions_graph.py` [CRÉER]

## Modèle de Données

Le graphe est persisté dans `.workflow/decisions-graph.json` :

```json
{
  "schema_version": "1.0.0",
  "nodes": [
    {
      "id": "DEC-001",
      "task_id": "TASK-004",
      "summary": "ORM : Prisma",
      "reason": "Meilleure DX, migrations plus fiables sur PostgreSQL",
      "scope": "data-layer",
      "confidence": "HIGH",
      "date": "2026-03-12",
      "tags": ["orm", "database", "prisma"]
    },
    {
      "id": "DEC-002",
      "task_id": "TASK-008",
      "summary": "Auth : JWT uniquement",
      "reason": "Architecture stateless requise pour serverless",
      "scope": "auth",
      "confidence": "HIGH",
      "date": "2026-03-18",
      "tags": ["auth", "jwt"]
    }
  ],
  "edges": [
    {
      "from": "DEC-002",
      "to": "DEC-005",
      "type": "DEPENDS_ON",
      "note": "JWT signing key dépend de la config secrets"
    },
    {
      "from": "DEC-007",
      "to": "DEC-001",
      "type": "SUPERSEDES",
      "note": "Migration vers Drizzle (Prisma trop lourd en serverless)"
    }
  ]
}
```

## Types de Relations

| Type | Sens | Exemple |
|------|------|---------|
| `DEPENDS_ON` | A nécessite que B reste vrai | "JWT auth dépend de secrets manager" |
| `CONTRADICTS` | A et B sont incompatibles | "REST API contredit GraphQL-only" |
| `SUPERSEDES` | A remplace B (B devient obsolète) | "Drizzle supersede Prisma" |
| `REFINES` | A précise/contraint B | "Postgres 16 refine 'database = postgres'" |
| `SCOPED_BY` | A est limitée au scope de B | "Bcrypt scoped_by auth" |

## Niveaux de Confiance

```
HIGH    : décision explicite de l'utilisateur, validée
MEDIUM  : décision LLM annotée DÉCISION:/RAISON: dans le code
LOW     : décision inférée par observation (ex : import vu dans le code)
```

Les décisions `LOW` ne déclenchent pas de rétro-propagation automatique — uniquement un avertissement.

## Implémentation

```python
# src/workflow/core/decisions_graph.py
import json
from pathlib import Path
from datetime import datetime, timezone
from typing import Literal

from workflow.tools.filesystem import FileSystem
from workflow.llm.llm_provider import LLMProvider

EdgeType = Literal['DEPENDS_ON', 'CONTRADICTS', 'SUPERSEDES', 'REFINES', 'SCOPED_BY']
Confidence = Literal['HIGH', 'MEDIUM', 'LOW']

GRAPH_FILE = 'decisions-graph.json'


class DecisionsGraph:
    def __init__(self, project_root: str, llm: LLMProvider | None = None):
        self.project_root = Path(project_root)
        self.fs = FileSystem(project_root)
        self.llm = llm
        self._cache: dict | None = None

    async def _load(self) -> dict:
        if self._cache is not None:
            return self._cache
        data = await self.fs.read_json(GRAPH_FILE)
        self._cache = data or {'schema_version': '1.0.0', 'nodes': [], 'edges': []}
        return self._cache

    async def _save(self, data: dict):
        await self.fs.write_json_atomic(GRAPH_FILE, data)
        self._cache = data

    async def add_decision(
        self,
        task_id: str,
        summary: str,
        reason: str,
        scope: str = 'global',
        confidence: Confidence = 'MEDIUM',
        tags: list[str] | None = None,
    ) -> dict:
        """Ajouter un nœud + détecter automatiquement les contradictions"""
        graph = await self._load()
        node_id = f'DEC-{len(graph["nodes"]) + 1:03d}'
        node = {
            'id': node_id,
            'task_id': task_id,
            'summary': summary,
            'reason': reason,
            'scope': scope,
            'confidence': confidence,
            'date': datetime.now(timezone.utc).date().isoformat(),
            'tags': tags or [],
        }
        graph['nodes'].append(node)

        # Détecter les contradictions automatiquement
        contradictions = await self.detect_contradictions(node, graph)
        for contra in contradictions:
            graph['edges'].append({
                'from': node_id,
                'to': contra['id'],
                'type': 'CONTRADICTS',
                'note': contra['reason'],
                'auto_detected': True,
            })

        await self._save(graph)
        return {'node': node, 'contradictions': contradictions}

    async def detect_contradictions(self, new_node: dict, graph: dict) -> list[dict]:
        """
        Détecter les contradictions entre une nouvelle décision et l'existant.
        Stratégie : (1) match par tag/scope, (2) si match, demander au LLM si contradictoire.
        """
        candidates = [
            n for n in graph['nodes']
            if n['scope'] == new_node['scope']
            or set(n.get('tags', [])) & set(new_node.get('tags', []))
        ]
        if not candidates or self.llm is None:
            return []

        # Lot de vérifications via LLM (role='fast')
        contradictions = []
        for candidate in candidates:
            prompt = f"""Décision A : {new_node['summary']} ({new_node['reason']})
Décision B : {candidate['summary']} ({candidate['reason']})

Ces deux décisions sont-elles techniquement incompatibles dans un même projet ?
Réponds UNIQUEMENT en JSON : {{"contradicts": true|false, "reason": "..."}}"""

            response = await self.llm.ask(prompt, role='fast', max_tokens=200)
            try:
                result = json.loads(response.strip())
                if result.get('contradicts'):
                    contradictions.append({
                        'id': candidate['id'],
                        'reason': result.get('reason', ''),
                    })
            except json.JSONDecodeError:
                continue

        return contradictions

    async def add_edge(self, from_id: str, to_id: str, edge_type: EdgeType, note: str = ''):
        """Ajouter une relation explicite entre deux décisions"""
        graph = await self._load()
        graph['edges'].append({
            'from': from_id,
            'to': to_id,
            'type': edge_type,
            'note': note,
            'auto_detected': False,
        })
        await self._save(graph)

    async def get_impacted_tasks(self, decision_id: str, task_manager) -> list[str]:
        """
        Trouver les tâches en cours (pending/in_progress) impactées par une décision.
        Stratégie : tâches dont le scope ou les tags overlappent avec ceux de la décision.
        """
        graph = await self._load()
        node = next((n for n in graph['nodes'] if n['id'] == decision_id), None)
        if not node:
            return []

        # Récupérer les tâches pending de la version active
        pending = await task_manager.get_pending_tasks_with_metadata()
        impacted = []
        for task in pending:
            task_tags = set(task.get('tags', []))
            if (
                task.get('scope') == node['scope']
                or task_tags & set(node.get('tags', []))
            ):
                impacted.append(task['id'])
        return impacted

    async def propagate_change(
        self, decision_id: str, change_note: str, task_manager
    ) -> dict:
        """
        Rétro-propager un changement de décision aux tâches en cours.
        Annoter chaque tâche impactée dans son Journal.
        """
        impacted = await self.get_impacted_tasks(decision_id, task_manager)
        for task_id in impacted:
            await task_manager.append_to_journal(
                task_id,
                f'[{datetime.now(timezone.utc).date().isoformat()}] '
                f'Décision {decision_id} mise à jour : {change_note}',
            )
        return {'decision': decision_id, 'impacted_tasks': impacted}

    async def get_node(self, node_id: str) -> dict | None:
        graph = await self._load()
        return next((n for n in graph['nodes'] if n['id'] == node_id), None)

    async def get_neighbors(
        self, node_id: str, edge_type: EdgeType | None = None
    ) -> list[dict]:
        """Retourner les voisins d'un nœud (optionnellement filtrés par type d'arête)"""
        graph = await self._load()
        result = []
        for edge in graph['edges']:
            if edge['from'] == node_id or edge['to'] == node_id:
                if edge_type and edge['type'] != edge_type:
                    continue
                other_id = edge['to'] if edge['from'] == node_id else edge['from']
                neighbor = next((n for n in graph['nodes'] if n['id'] == other_id), None)
                if neighbor:
                    result.append({**neighbor, 'edge_type': edge['type']})
        return result

    async def find_active_contradictions(self) -> list[dict]:
        """
        Lister toutes les contradictions actives entre décisions HIGH-confidence.
        Utilisé par WorkflowAgent au boot pour alerter l'utilisateur.
        """
        graph = await self._load()
        contradictions = []
        for edge in graph['edges']:
            if edge['type'] != 'CONTRADICTS':
                continue
            from_node = next((n for n in graph['nodes'] if n['id'] == edge['from']), None)
            to_node = next((n for n in graph['nodes'] if n['id'] == edge['to']), None)
            if not (from_node and to_node):
                continue
            if from_node['confidence'] == 'HIGH' and to_node['confidence'] == 'HIGH':
                contradictions.append({
                    'edge': edge,
                    'from': from_node,
                    'to': to_node,
                })
        return contradictions

    async def stats(self) -> dict:
        """Statistiques pour le briefing DaemonHeartbeat"""
        graph = await self._load()
        nodes = graph['nodes']
        edges = graph['edges']
        return {
            'total_decisions': len(nodes),
            'high_confidence': sum(1 for n in nodes if n['confidence'] == 'HIGH'),
            'total_relations': len(edges),
            'contradictions': sum(1 for e in edges if e['type'] == 'CONTRADICTS'),
            'superseded': sum(1 for e in edges if e['type'] == 'SUPERSEDES'),
        }
```

## Intégration avec les autres modules

**`ExecutionLoop` (Phase 3)** — chaque décision extraite (`DÉCISION:/RAISON:`) passe par `add_decision()` :

```python
async def _extract_decisions(self, task_id: str, raw: str):
    for match in DECISION_PATTERN.finditer(raw):
        result = await self.graph.add_decision(
            task_id=task_id,
            summary=match.group(1).strip(),
            reason=match.group(2).strip(),
            scope=self._infer_scope(task_id),
            confidence='MEDIUM',
        )
        if result['contradictions']:
            self.io.print_warning(
                f'⚠️  Cette décision contredit : '
                f'{", ".join(c["id"] for c in result["contradictions"])}'
            )
```

**`ContextManager._load_scored_decisions()` (Phase 2)** — utilise le graphe pour booster le score des décisions liées à la tâche courante via les arêtes `SCOPED_BY`/`REFINES`.

**`SyncChecker.check()` (Phase 1.8)** — au boot, appelle `find_active_contradictions()` et alerte si > 0.

**`MCPServer` (Phase 6)** — expose `workflow_get_decision_graph()` et `workflow_log_decision()` qui s'appuient sur cette classe.

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `add_decision()` ajoute un nœud avec ID auto-généré `DEC-XXX` | ⬜ |
| 2 | `add_decision()` détecte automatiquement les contradictions via LLM (mocké en test) | ⬜ |
| 3 | `add_edge()` permet d'ajouter manuellement une relation typée | ⬜ |
| 4 | `get_impacted_tasks()` retourne les tâches `pending` matchant scope/tags | ⬜ |
| 5 | `propagate_change()` annote le Journal de chaque tâche impactée | ⬜ |
| 6 | `find_active_contradictions()` ne retourne que les contradictions entre `HIGH` | ⬜ |
| 7 | `get_neighbors(id, edge_type='SUPERSEDES')` filtre correctement | ⬜ |
| 8 | Le graphe est persisté en JSON validé contre `decisions-graph.schema.json` | ⬜ |
| 9 | Cache mémoire évite les relectures répétées du fichier | ⬜ |
| 10 | Tests : ajout nœud, détection contradiction (mock LLM), rétro-propagation, stats | ⬜ |

## Notes d'Implémentation

- La détection automatique de contradictions a un coût LLM. Limiter à `role='fast'` et batcher si possible.
- Si le graphe dépasse ~500 nœuds, prévoir un index in-memory par scope/tag pour `get_impacted_tasks` (Phase 9 si nécessaire).
- Les arêtes `auto_detected: true` peuvent être révoquées par l'utilisateur via `MCPServer.workflow_remove_edge(from, to)`.
