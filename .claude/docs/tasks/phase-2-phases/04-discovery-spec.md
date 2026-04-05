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

// Styles de design proposés à l'utilisateur
const DESIGN_STYLES = [
  { key: 'minimaliste',    label: 'Minimaliste',       desc: 'épuré, beaucoup d\'espace blanc, peu de couleurs' },
  { key: 'material',       label: 'Material Design',   desc: 'composants Google, couleurs vives, élévations' },
  { key: 'glassmorphism',  label: 'Glassmorphism',     desc: 'flou de fond, transparences, effet vitré' },
  { key: 'neomorphisme',   label: 'Néomorphisme',      desc: 'relief doux, monochrome, ombres subtiles' },
  { key: 'brutaliste',     label: 'Brutaliste',        desc: 'contraste élevé, typographie forte, sans fioritures' },
  { key: 'doux',           label: 'Doux / Pastel',     desc: 'couleurs douces, coins arrondis, friendly' },
  { key: 'dashboard-pro',  label: 'Dashboard Pro',     desc: 'dense, data-centric, sidebar, pro tools' },
  { key: 'mobile-first',   label: 'Mobile-First',      desc: 'gros boutons, bottom navigation, tactile' },
  { key: 'cyberpunk',      label: 'Cyberpunk',         desc: 'sombre, néons, futuriste, high contrast' },
  { key: 'personnalise',   label: 'Personnalisé',      desc: 'tu décris exactement ce que tu veux' },
];

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
    } else {
      // 6. Sauvegarder la vision
      await this.memory.saveVision(visionContent);
      this.io.display('✅ vision.md sauvegardé.');
    }

    // 7. Collecter les préférences design (obligatoire si le projet a une interface)
    await this._collectDesignPreferences();

    return { completed: true, output: 'vision.md' };
  }

  // Collecter le style de design souhaité par l'utilisateur
  // Sauvegarde dans .workflow/design.json — utilisé par ValidationPhase pour générer les mockups
  async _collectDesignPreferences() {
    this.io.display('\n─────────────────────────────────────────');
    this.io.display('🎨  STYLE DE DESIGN');
    this.io.display('─────────────────────────────────────────');
    this.io.display('Quel style visuel veux-tu pour ton application ?\n');

    DESIGN_STYLES.forEach((s, i) => {
      this.io.display(`  ${String(i + 1).padStart(2)}. ${s.label.padEnd(18)} — ${s.desc}`);
    });

    const choice = await this.io.ask('\nTon choix (numéro ou nom) :');

    // Résoudre le choix — accepte numéro OU nom partiel (case-insensitive)
    const num = parseInt(choice, 10);
    let selected = null;
    if (num >= 1 && num <= DESIGN_STYLES.length) {
      selected = DESIGN_STYLES[num - 1];
    } else {
      selected = DESIGN_STYLES.find(s =>
        s.key.includes(choice.toLowerCase()) || s.label.toLowerCase().includes(choice.toLowerCase())
      ) ?? DESIGN_STYLES[DESIGN_STYLES.length - 1]; // fallback : personnalisé
    }

    this.io.display(`\n✓ Style sélectionné : ${selected.label}`);

    // Détails supplémentaires
    const colorScheme = await this.io.ask('Thème de couleurs ? (clair / sombre / les deux) :');
    const references = await this.io.ask('Applications dont tu aimes le design (optionnel, ex: Linear, Notion, Stripe) :');
    const customNotes = selected.key === 'personnalise'
      ? await this.io.ask('Décris précisément le style que tu veux :')
      : await this.io.ask('Autre précision sur le design ? (optionnel, Entrée pour passer) :');

    // Construire l'objet design
    const design = {
      style: selected.key,
      styleLabel: selected.label,
      colorScheme: colorScheme.trim() || 'clair',
      references: references.trim()
        ? references.split(/[,;]/).map(r => r.trim()).filter(Boolean)
        : [],
      customNotes: customNotes.trim() || null,
      collectedAt: new Date().toISOString(),
    };

    await this.memory.saveDesign(design);
    this.io.display('✅ design.json sauvegardé.');
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
| 2 | `DiscoveryPhase` recommence si l'utilisateur refuse la vision révisée | ⬜ |
| 3 | `SpecificationPhase` échoue proprement si `vision.md` est absent | ⬜ |
| 4 | `SpecificationPhase` parse le JSON et affiche un résumé lisible | ⬜ |
| 5 | Les corrections sont intégrées avant la validation finale | ⬜ |
| 6 | `_collectDesignPreferences()` est toujours appelé après la validation de `vision.md` | ⬜ |
| 7 | L'utilisateur peut choisir le style par numéro OU par nom (partiel, case-insensitive) | ⬜ |
| 8 | `design.json` est sauvegardé avec `style`, `colorScheme`, `references`, `customNotes` | ⬜ |
| 9 | Si l'utilisateur passe la question des références (Entrée vide), `references` est `[]` | ⬜ |
| 10 | Si le style choisi est "personnalisé", une question supplémentaire est posée pour la description | ⬜ |
