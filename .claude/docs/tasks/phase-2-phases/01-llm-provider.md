# Phase 2 — Tâche 2.1 : LLMProvider.js + PromptBuilder.js

## Objectif

Créer l'abstraction LLM. `LLMProvider` encapsule les appels à Claude API. `PromptBuilder` construit les prompts en injectant systématiquement le contexte projet — jamais de question posée au LLM sans contexte.

## Dépendances

- Phase 1 complète ✅

## Fichiers à Créer

- `src/llm/LLMProvider.js` [CRÉER]
- `src/llm/PromptBuilder.js` [CRÉER]
- `tests/unit/PromptBuilder.test.js` [CRÉER]

## Implémentation — `LLMProvider.js`

```javascript
// src/llm/LLMProvider.js
import Anthropic from '@anthropic-ai/sdk';

export class LLMProvider {
  constructor(config = {}) {
    this.client = new Anthropic({ apiKey: config.apiKey ?? process.env.ANTHROPIC_API_KEY });
    this.model = config.model ?? 'claude-sonnet-4-6';
    this.maxTokens = config.maxTokens ?? 8192;
  }

  // Appel simple — retourne le texte
  async ask(prompt, systemPrompt = null) {
    const messages = [{ role: 'user', content: prompt }];
    const params = {
      model: this.model,
      max_tokens: this.maxTokens,
      messages,
    };
    if (systemPrompt) params.system = systemPrompt;

    const response = await this.client.messages.create(params);
    return response.content[0].text;
  }

  // Conversation multi-tours (pour les phases interactives)
  async chat(history, systemPrompt = null) {
    const params = {
      model: this.model,
      max_tokens: this.maxTokens,
      messages: history,
    };
    if (systemPrompt) params.system = systemPrompt;

    const response = await this.client.messages.create(params);
    return response.content[0].text;
  }

  // Streaming (pour affichage progressif en CLI)
  async stream(prompt, systemPrompt = null, onChunk) {
    const stream = this.client.messages.stream({
      model: this.model,
      max_tokens: this.maxTokens,
      system: systemPrompt,
      messages: [{ role: 'user', content: prompt }],
    });

    for await (const chunk of stream) {
      if (chunk.type === 'content_block_delta') {
        onChunk(chunk.delta.text);
      }
    }

    return stream.finalMessage();
  }
}
```

## Implémentation — `PromptBuilder.js`

```javascript
// src/llm/PromptBuilder.js

export class PromptBuilder {
  // Prompt système global de Workflow
  static systemPrompt(projectContext) {
    return `Tu es Workflow, un agent de code expert.
Tu travailles sur le projet "${projectContext.name}" (${projectContext.language}/${projectContext.framework}).
Version active : ${projectContext.currentVersion ?? 'aucune'}.

Règles absolues :
- Consulte toujours les décisions techniques passées avant de proposer une solution.
- Respecte la stack définie : ${JSON.stringify(projectContext)}.
- Une tâche = max 4h de travail = max 3 fichiers créés/modifiés.
- Jamais de commande shell non listée dans allowed_commands.`;
  }

  // Prompt pour la Discovery Phase
  static discovery(partialVision) {
    return `L'utilisateur décrit son idée. Pose 3 à 5 questions ciblées pour clarifier :
- Le problème résolu
- Les utilisateurs cibles
- Les fonctionnalités essentielles vs. optionnelles
- Les contraintes techniques éventuelles

Idée initiale :
${partialVision}

Pose tes questions de manière concise et directe.`;
  }

  // Prompt pour générer features.json depuis vision.md
  // Chaque feature inclut un champ "intent" propagé aux tâches par ValidationPhase
  static specFeatures(vision, existingFeatures = null) {
    return `À partir de cette vision produit, génère une liste de fonctionnalités structurée par version.
Retourne un JSON valide avec la structure : { "v1.0": [{ "id": "F001", "name": "...", "description": "...", "priority": "HIGH|MEDIUM|LOW", "intent": "pourquoi l'utilisateur veut vraiment cette fonctionnalité" }], "v1.5": [...] }

Vision :
${vision}

${existingFeatures ? `Fonctionnalités existantes à affiner :\n${JSON.stringify(existingFeatures, null, 2)}` : ''}

Retourne UNIQUEMENT le JSON, sans explication.`;
  }

  // Prompt pour générer les tâches d'une version
  static generateTasks(version, features, techStack, existingTaskIds) {
    return `Génère les fichiers de tâches pour la version ${version}.

Fonctionnalités à implémenter :
${JSON.stringify(features[version], null, 2)}

Stack technique :
${JSON.stringify(techStack, null, 2)}

Tâches déjà créées : ${existingTaskIds.join(', ') || 'aucune'}

RÈGLES OBLIGATOIRES :
1. Si c'est la v1.0, les deux premières tâches doivent être TASK-001 (setup + linter) et TASK-002 (tests + smoke test).
2. Chaque tâche = max 4h de travail = max 3 fichiers créés/modifiés.
3. Si une fonctionnalité nécessite plus, découpe-la en sous-tâches.
4. Chaque tâche doit être auto-suffisante (contexte, user story, dépendances, fichiers, critères).

Retourne un tableau JSON de tâches avec la structure :
[{ "id": "TASK-XXX", "title": "...", "context": "...", "userStory": "...", "dependencies": [...], "files": [...], "criteria": [...] }]

Retourne UNIQUEMENT le JSON.`;
  }

  // Prompt pour générer du code pour une tâche
  static generateCode(task, relevantFiles, relevantDecisions) {
    const decisionsText = relevantDecisions.length > 0
      ? `\nDécisions techniques précédentes pertinentes :\n${relevantDecisions.map(d => d.summary).join('\n')}`
      : '';

    const filesText = Object.entries(relevantFiles)
      .filter(([, content]) => content !== null)
      .map(([path, content]) => `\n### ${path}\n\`\`\`\n${content}\n\`\`\``)
      .join('');

    // L'intent est injecté AVANT les critères d'acceptation
    const intentText = task.intent
      ? `\n## Intent (pourquoi cette fonctionnalité)\n${task.intent}`
      : '';

    return `Implémente la tâche suivante :

# ${task.id} : ${task.title}

## User Story
${task.userStory}
${intentText}

## Fichiers à créer/modifier
${task.files?.map(f => `- ${f.path} [${f.action}]`).join('\n')}

## Critères d'acceptation
${task.criteria?.map(c => `- ${c}`).join('\n')}
${decisionsText}

## Fichiers existants pertinents
${filesText || '(aucun fichier existant pour cette tâche)'}

Si tu prends une décision technique importante, annote-la ainsi dans ta réponse :
DÉCISION: <la décision>
RAISON: <la justification>

## Format de réponse OBLIGATOIRE

Pour chaque fichier à créer ou modifier, utilise EXACTEMENT ce format :

### chemin/vers/fichier.js
\`\`\`javascript
[contenu du fichier]
\`\`\`

Ne jamais utiliser un autre format. Le parser de Workflow extrait les fichiers
depuis ce format précis — tout autre format est ignoré silencieusement.

Génère le code complet pour chaque fichier. Commence directement par le code.`;
  }

  // Prompt de retry — utilisé dès la tentative 2, injecte l'erreur précédente
  static generateCodeRetry(task, relevantFiles, relevantDecisions, lastError, knownFix) {
    const base = PromptBuilder.generateCode(task, relevantFiles, relevantDecisions);
    const errorSection = `\n\n---\nTa tentative précédente a échoué avec cette erreur :\n\`\`\`\n${lastError.output}\n\`\`\``;
    const fixSection = knownFix ? `\nUn fix connu pour ce type d'erreur est : ${knownFix}` : '';
    return base + errorSection + fixSection + '\n\nCorrige le code en tenant compte de cette erreur.';
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `LLMProvider.ask()` appelle l'API et retourne le texte | ⬜ |
| 2 | `LLMProvider.stream()` appelle `onChunk` pour chaque token | ⬜ |
| 3 | `PromptBuilder.systemPrompt()` inclut le nom du projet et la stack | ⬜ |
| 4 | `PromptBuilder.generateTasks()` inclut la règle de granularité | ⬜ |
| 5 | `PromptBuilder.generateCode()` injecte `task.intent` avant les critères d'acceptation | ⬜ |
| 6 | `PromptBuilder.generateCode()` inclut les annotations DÉCISION/RAISON dans le prompt | ⬜ |
| 7 | `PromptBuilder.generateCode()` inclut l'instruction de format `### chemin/fichier.js` | ⬜ |
| 8 | `PromptBuilder.generateCodeRetry()` inclut `lastError.output` dans le prompt | ⬜ |
| 9 | `PromptBuilder.generateCodeRetry()` inclut `knownFix` si non null | ⬜ |
| 10 | Tests unitaires mockent l'API Claude | ⬜ |
