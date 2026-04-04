# Phase 2 — Tâche 2.4 : DiscoveryPhase + SpecificationPhase

## Objectif

Implémenter les deux premières phases interactives. `DiscoveryPhase` transforme une idée floue en `vision.md` structurée. `SpecificationPhase` transforme la vision en `features.json` validé.

## Dépendances

- Tâches 2.1, 2.2, 2.3 ✅

## Fichiers à Créer

- `src/phases/DiscoveryPhase.js` [CRÉER]
- `src/phases/SpecificationPhase.js` [CRÉER]

## `DiscoveryPhase.js`

```javascript
// src/phases/DiscoveryPhase.js
import { ProjectMemory } from '../core/ProjectMemory.js';
import { PromptBuilder } from '../llm/PromptBuilder.js';

export class DiscoveryPhase {
  constructor(projectRoot, llm, io) {
    this.memory = new ProjectMemory(projectRoot);
    this.llm = llm;
    this.io = io;
  }

  async run() {
    // 1. Demander la description initiale
    const initialDescription = await this.io.ask(
      'Décris ton projet en quelques phrases (ce que ça fait, pour qui, quel problème ça résout) :'
    );

    // 2. LLM pose des questions ciblées
    const questionsPrompt = PromptBuilder.discovery(initialDescription);
    const questions = await this.llm.ask(questionsPrompt);
    this.io.display(questions);

    // 3. Collecter les réponses
    const answers = await this.io.ask('Tes réponses :');

    // 4. LLM génère vision.md
    // L'intent global (pourquoi l'utilisateur construit ce projet) est capturé ici
    const visionPrompt = `Sur la base de cette description et de ces réponses, génère un fichier vision.md structuré.

Description initiale : ${initialDescription}
Questions posées : ${questions}
Réponses : ${answers}

Le vision.md doit couvrir : le problème résolu, les utilisateurs cibles, les fonctionnalités clés, les contraintes techniques, ce qui est EXCLU de la v1.
Inclure une section "## Intent Global" qui capture le "pourquoi profond" du projet — ce que l'utilisateur veut vraiment accomplir au-delà des fonctionnalités listées.`;

    const visionContent = await this.llm.ask(visionPrompt);

    // 5. Validation par l'utilisateur
    this.io.display('\n--- VISION GÉNÉRÉE ---\n' + visionContent);
    const approved = await this.io.confirm('Cette vision te convient ? (o/n)');

    if (!approved) {
      const corrections = await this.io.ask('Quelles corrections apporter ?');
      // Régénérer en injectant les corrections dans le prompt — elles ne sont pas perdues
      const correctedPrompt = `${visionPrompt}\n\nCorrections demandées par l'utilisateur : ${corrections}\n\nGénère une vision.md révisée tenant compte de ces corrections.`;
      const correctedVision = await this.llm.ask(correctedPrompt);

      this.io.display('\n--- VISION RÉVISÉE ---\n' + correctedVision);
      const finalApproved = await this.io.confirm('Cette version révisée te convient ? (o/n)');

      if (!finalApproved) return this.run(); // Recommence depuis zéro si toujours rejeté

      await this.memory.saveVision(correctedVision);
      this.io.display('✅ vision.md sauvegardé.');
      return { completed: true, output: 'vision.md' };
    }

    // 6. Sauvegarder
    await this.memory.saveVision(visionContent);
    this.io.display('✅ vision.md sauvegardé.');

    return { completed: true, output: 'vision.md' };
  }
}
```

## `SpecificationPhase.js`

```javascript
// src/phases/SpecificationPhase.js
import { ProjectMemory } from '../core/ProjectMemory.js';
import { PromptBuilder } from '../llm/PromptBuilder.js';

export class SpecificationPhase {
  constructor(projectRoot, llm, io) {
    this.memory = new ProjectMemory(projectRoot);
    this.llm = llm;
    this.io = io;
  }

  async run() {
    const vision = await this.memory.getVision();
    if (!vision) throw new Error('vision.md manquant — relancer DiscoveryPhase');

    // 1. LLM génère les fonctionnalités
    // Chaque fonctionnalité inclut un champ "intent" (pourquoi l'utilisateur la veut)
    // Cet intent sera propagé dans les tâches générées par ValidationPhase
    const featuresPrompt = PromptBuilder.specFeatures(vision);
    const featuresJson = await this.llm.ask(featuresPrompt);

    let features;
    try {
      features = JSON.parse(featuresJson);
    } catch {
      throw new Error('LLM a retourné un JSON invalide pour features.json');
    }

    // 2. Afficher pour validation
    this.io.display('\n--- FONCTIONNALITÉS PROPOSÉES ---');
    for (const [version, feats] of Object.entries(features)) {
      this.io.display(`\n${version} :`);
      feats.forEach(f => this.io.display(`  [${f.priority}] ${f.name} — ${f.description}`));
    }

    const approved = await this.io.confirm('\nCes fonctionnalités te conviennent ? (o/n)');

    if (!approved) {
      const corrections = await this.io.ask('Quelles modifications ? (ex: retire F003, ajoute export PDF en v1.5)');
      const correctedPrompt = PromptBuilder.specFeatures(vision, features) +
        `\n\nModifications demandées : ${corrections}`;
      const corrected = await this.llm.ask(correctedPrompt);
      try {
        features = JSON.parse(corrected);
      } catch {
        throw new Error('LLM a retourné un JSON invalide après corrections — relancer la spécification');
      }
    }

    // 3. Sauvegarder
    await this.memory.saveFeatures(features);
    this.io.display('✅ features.json sauvegardé.');

    return { completed: true, output: 'features.json' };
  }
}
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `DiscoveryPhase.run()` sauvegarde `vision.md` après approbation | ⬜ |
| 2 | `DiscoveryPhase` recommence si l'utilisateur refuse la vision | ⬜ |
| 3 | `SpecificationPhase` échoue proprement si `vision.md` est absent | ⬜ |
| 4 | `SpecificationPhase` parse le JSON et affiche un résumé lisible | ⬜ |
| 5 | Les corrections sont intégrées avant la validation finale | ⬜ |
