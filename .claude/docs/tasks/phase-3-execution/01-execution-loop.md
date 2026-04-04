# Phase 3 — Tâche 3.1 : ExecutionLoop.js

## Objectif

Implémenter la boucle d'auto-correction. `ExecutionLoop` génère le code via LLM, l'applique, lance `build_validate`, lance les tests, et se corrige automatiquement jusqu'à 3 fois avant d'escalader.

## Dépendances

- Phase 2 ✅

## Fichiers à Créer

- `src/tools/ExecutionLoop.js` [CRÉER]
- `tests/unit/ExecutionLoop.test.js` [CRÉER]

## Implémentation

```javascript
// src/tools/ExecutionLoop.js
import { exec } from 'child_process';
import { promisify } from 'util';
import { writeFile, mkdir } from 'fs/promises';
import { dirname, join } from 'path';
import { PromptBuilder } from '../llm/PromptBuilder.js';
import { FailurePatterns } from './FailurePatterns.js';

const execAsync = promisify(exec);
const MAX_ATTEMPTS = 3;

export class ExecutionLoop {
  constructor(projectRoot, llm, techStack) {
    this.projectRoot = projectRoot;
    this.llm = llm;
    this.techStack = techStack;
    this.failurePatterns = new FailurePatterns(projectRoot);
  }

  async run(task, ctx) {
    let attempts = 0;
    let lastError = null;
    const errorHistory = []; // Historique des erreurs pour détecter les boucles

    while (attempts < MAX_ATTEMPTS) {
      attempts++;

      try {
        // 1. Générer le code (avec contexte d'erreur pour les tentatives 2 et 3)
        const codePrompt = attempts === 1
          ? PromptBuilder.generateCode(task, ctx.task.relevantFiles, ctx.task.relevantDecisions)
          : PromptBuilder.generateCodeRetry(task, ctx.task.relevantFiles, ctx.task.relevantDecisions, lastError, ctx.knownFix ?? null);
        const codeResponse = await this.llm.ask(codePrompt);

        // 2. Parser et appliquer les fichiers générés
        const files = this.parseCodeResponse(codeResponse);
        await this.applyFiles(files);

        // 3. Build + validate
        const buildResult = await this.runCommand(this.techStack.build_validate);
        if (!buildResult.success) {
          lastError = { type: 'BUILD', output: buildResult.output, attempt: attempts };
          errorHistory.push(buildResult.output);

          // Vérifier si on tourne en boucle sur la même erreur
          if (this._isLooping(errorHistory)) {
            return {
              success: false,
              escalate: true,
              reason: 'LOOP_DETECTED',
              escalationContext: this.buildEscalationContext(task, lastError, attempts),
            };
          }

          // Chercher un pattern connu avant de relancer le LLM
          const knownFix = await this.failurePatterns.match(buildResult.output);
          if (knownFix) {
            ctx.knownFix = knownFix; // Injecté dans le prochain prompt
          }
          continue;
        }

        // 4. Tests
        const testResult = await this.runCommand(this.techStack.test);
        if (!testResult.success) {
          lastError = { type: 'TEST', output: testResult.output, attempt: attempts };
          errorHistory.push(testResult.output);

          if (this._isLooping(errorHistory)) {
            return {
              success: false,
              escalate: true,
              reason: 'LOOP_DETECTED',
              escalationContext: this.buildEscalationContext(task, lastError, attempts),
            };
          }

          const knownFix = await this.failurePatterns.match(testResult.output);
          if (knownFix) ctx.knownFix = knownFix;
          continue;
        }

        // Succès — extraire les décisions prises + apprendre des échecs
        const decisions = this.extractDecisions(codeResponse);
        if (errorHistory.length > 0) {
          await this.failurePatterns.learnFromSuccess(errorHistory, codeResponse);
        }
        return { success: true, decisions };

      } catch (err) {
        lastError = { type: 'EXCEPTION', output: err.message, attempt: attempts };
      }
    }

    // Échec après MAX_ATTEMPTS
    return {
      success: false,
      escalate: true,
      escalationContext: this.buildEscalationContext(task, lastError, attempts),
    };
  }

  // Détecter si les erreurs successives sont identiques (boucle stérile)
  _isLooping(errorHistory) {
    if (errorHistory.length < 2) return false;
    const last = errorHistory[errorHistory.length - 1];
    const prev = errorHistory[errorHistory.length - 2];
    // Similarité simple : 80% de chevauchement de tokens
    const lastWords = new Set(last.split(/\s+/).slice(0, 50));
    const prevWords = prev.split(/\s+/).slice(0, 50).filter(w => lastWords.has(w));
    return prevWords.length / lastWords.size > 0.8;
  }

  // Exécuter une commande — seulement si dans allowed_commands
  // Correspondance exacte par défaut ; wildcard explicite avec "*" en fin de pattern
  async runCommand(cmd) {
    const allowed = this.techStack.allowed_commands ?? [];
    const isAllowed = allowed.some(pattern => {
      if (pattern.endsWith('*')) {
        // Wildcard explicite : "npm run *" autorise toutes les sous-commandes npm run
        return cmd.startsWith(pattern.slice(0, -1));
      }
      return cmd === pattern; // Correspondance exacte par défaut
    });

    if (!isAllowed) {
      throw new Error(
        `Commande non autorisée : "${cmd}"\n` +
        `Commandes autorisées : ${allowed.join(', ')}\n` +
        `Pour autoriser cette commande, l'ajouter à tech-stack.json#allowed_commands.`
      );
    }

    try {
      const { stdout, stderr } = await execAsync(cmd, {
        cwd: this.projectRoot,
        timeout: 60000,
      });
      return { success: true, output: stdout + stderr };
    } catch (err) {
      return { success: false, output: err.stdout + err.stderr + err.message };
    }
  }

  // Parser la réponse LLM pour extraire les fichiers à créer
  //
  // Format imposé au LLM dans le prompt (via PromptBuilder.generateCode) :
  // Pour chaque fichier à créer ou modifier, le LLM DOIT utiliser EXACTEMENT ce format :
  //
  //   ### chemin/vers/fichier.js
  //   ```javascript
  //   [contenu du fichier]
  //   ```
  //
  // Tout autre format est ignoré silencieusement. Le prompt doit explicitement
  // instruire le LLM d'utiliser ce format — voir PromptBuilder.generateCode().
  parseCodeResponse(response) {
    const files = [];
    const regex = /^### ([^\n]+)\n```(?:\w+)?\n([\s\S]+?)```/gm;
    let match;
    while ((match = regex.exec(response)) !== null) {
      files.push({ path: match[1].trim(), content: match[2] });
    }
    if (files.length === 0) {
      // Log avertissement — la réponse LLM n'a produit aucun fichier parseable
      console.warn('[ExecutionLoop] Aucun fichier extrait de la réponse LLM. Vérifier le format du prompt.');
    }
    return files;
  }

  // Appliquer les fichiers générés sur le disque
  async applyFiles(files) {
    for (const file of files) {
      const fullPath = join(this.projectRoot, file.path);
      await mkdir(dirname(fullPath), { recursive: true });
      await writeFile(fullPath, file.content, 'utf-8');
    }
  }

  // Extraire les décisions techniques de la réponse LLM
  // Le LLM est guidé pour les annoter avec "DÉCISION:" dans sa réponse
  extractDecisions(response) {
    const decisions = [];
    const regex = /DÉCISION:\s*(.+)\nRAISON:\s*(.+)/g;
    let match;
    while ((match = regex.exec(response)) !== null) {
      decisions.push({ decision: match[1].trim(), reason: match[2].trim() });
    }
    return decisions;
  }

  buildEscalationContext(task, lastError, attempts) {
    return `Tâche : ${task.id} — ${task.title}
Tentatives : ${attempts}/${MAX_ATTEMPTS}
Type d'erreur : ${lastError?.type}
Dernière erreur :
${lastError?.output ?? '(aucune sortie)'}

Critères non validés :
${task.criteria?.map(c => `  - ${c}`).join('\n') ?? '(aucun)'}`;
  }
}
```

## PromptBuilder — `generateCodeRetry`

`PromptBuilder.generateCodeRetry(task, files, decisions, lastError, knownFix)` est utilisé dès la tentative 2. Il inclut :

- Le contenu complet de `lastError.output` (sortie d'erreur complète)
- Le fix connu si `knownFix` n'est pas null
- Formulation type : "Ta tentative précédente a échoué avec cette erreur : [lastError.output]. [Si knownFix : Un fix connu pour ce type d'erreur est : knownFix]. Corrige le code."

```javascript
// Dans src/llm/PromptBuilder.js — à ajouter
static generateCodeRetry(task, relevantFiles, relevantDecisions, lastError, knownFix) {
  const base = PromptBuilder.generateCode(task, relevantFiles, relevantDecisions);
  const errorSection = `\n\n---\nTa tentative précédente a échoué avec cette erreur :\n\`\`\`\n${lastError.output}\n\`\`\``;
  const fixSection = knownFix ? `\nUn fix connu pour ce type d'erreur est : ${knownFix}` : '';
  return base + errorSection + fixSection + '\n\nCorrige le code en tenant compte de cette erreur.';
}
```

## Failure Patterns — `FailurePatterns.js`

Module dédié à la mémoire d'échecs, stockée dans `.workflow/failure-patterns.json`.

```javascript
// src/tools/FailurePatterns.js
import { FileSystem } from './FileSystem.js';

export class FailurePatterns {
  constructor(projectRoot) {
    this.fs = new FileSystem(projectRoot);
    this.patternsPath = this.fs.paths.failurePatterns();
  }

  // Chercher un pattern connu correspondant à une erreur
  // Retourne le fix documenté, ou null si aucun pattern trouvé
  async match(errorOutput) {
    const patterns = await this.fs.readJSON(this.patternsPath) ?? [];
    for (const p of patterns) {
      if (errorOutput.includes(p.fingerprint)) {
        p.occurrences++;
        p.lastSeen = new Date().toISOString().split('T')[0];
        await this._save(patterns);
        return p.fix;
      }
    }
    return null;
  }

  // Apprendre depuis un succès après des échecs
  async learnFromSuccess(errorHistory, successResponse) {
    if (!errorHistory.length) return;
    const fingerprint = this._extractFingerprint(errorHistory[errorHistory.length - 1]);
    const fix = this._extractFix(successResponse);
    if (!fingerprint || !fix) return;

    const patterns = await this.fs.readJSON(this.patternsPath) ?? [];
    const existing = patterns.find(p => p.fingerprint === fingerprint);
    if (existing) {
      existing.fix = fix;
      existing.occurrences++;
      existing.lastSeen = new Date().toISOString().split('T')[0];
    } else {
      patterns.push({
        fingerprint,
        fix,
        occurrences: 1,
        learnedAt: new Date().toISOString().split('T')[0],
        lastSeen: new Date().toISOString().split('T')[0],
      });
    }
    await this._save(patterns);
  }

  // Extraire une empreinte stable depuis un message d'erreur
  // Prend les 10 premiers mots significatifs (ignore les chemins et numéros de ligne)
  _extractFingerprint(errorOutput) {
    if (!errorOutput) return null;
    const cleaned = errorOutput
      .replace(/\/[\w./-]+/g, '<path>')       // Remplacer les chemins
      .replace(/:\d+:\d+/g, ':<line>')        // Remplacer les numéros de ligne
      .replace(/\s+/g, ' ')
      .trim();
    // Prendre les 12 premiers mots comme empreinte
    const words = cleaned.split(' ').slice(0, 12).join(' ');
    return words.length > 5 ? words : null;
  }

  // Extraire le fix appliqué depuis la réponse LLM réussie
  // Le LLM est guidé pour annoter les corrections avec "FIX:" dans sa réponse
  _extractFix(successResponse) {
    const match = successResponse?.match(/FIX:\s*(.+)/);
    return match ? match[1].trim() : null;
  }

  async _save(patterns) {
    await this.fs.writeJSON(this.patternsPath, patterns);
  }
}
```

**Format `failure-patterns.json`** :
```json
[
  {
    "fingerprint": "Cannot find module",
    "fix": "Ajouter type:module dans package.json et vérifier les imports avec extension .js",
    "occurrences": 3,
    "lastSeen": "2025-03-18",
    "learnedAt": "2025-03-12"
  }
]
```

Les patterns sont **cross-tâches** dans un projet mais pas encore cross-projets (ça, c'est la `WorkflowLibrary` en Phase 6).

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `runCommand` rejette toute commande non listée dans `allowed_commands` | ⬜ |
| 1b | `runCommand("npm install malicious-package")` est rejeté même si `"npm install"` est dans `allowed_commands` | ⬜ |
| 1c | `runCommand("npm run build:prod")` est accepté si `"npm run *"` est dans `allowed_commands` | ⬜ |
| 2 | La boucle retente exactement 3 fois avant d'escalader | ⬜ |
| 3 | `parseCodeResponse` extrait les fichiers des blocs de code au format `### chemin/fichier.js` | ⬜ |
| 3b | `parseCodeResponse` log un avertissement si aucun fichier n'est extrait | ⬜ |
| 4 | `applyFiles` crée les dossiers manquants automatiquement | ⬜ |
| 5 | `escalationContext` contient les erreurs et critères non validés | ⬜ |
| 6 | `_isLooping()` détecte quand deux erreurs consécutives sont identiques à 80% | ⬜ |
| 7 | La boucle escalade immédiatement avec `LOOP_DETECTED` si boucle détectée | ⬜ |
| 8 | `FailurePatterns.match()` retourne le fix documenté si l'erreur correspond | ⬜ |
| 9 | `FailurePatterns.learnFromSuccess()` enregistre un nouveau pattern après succès | ⬜ |
| 9b | `FailurePatterns._extractFingerprint()` remplace les chemins et numéros de ligne avant d'extraire l'empreinte | ⬜ |
| 9c | `FailurePatterns._extractFingerprint()` retourne `null` si l'output est vide ou trop court | ⬜ |
| 9d | `FailurePatterns._extractFix()` retourne `null` si la réponse LLM ne contient pas d'annotation `FIX:` | ⬜ |
| 10 | Tests unitaires mockent les commandes shell | ⬜ |
| 11 | `PromptBuilder.generateCodeRetry` inclut `lastError.output` dans le prompt de retry | ⬜ |
