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

const execAsync = promisify(exec);
const MAX_ATTEMPTS = 3;

export class ExecutionLoop {
  constructor(projectRoot, llm, techStack) {
    this.projectRoot = projectRoot;
    this.llm = llm;
    this.techStack = techStack;
  }

  async run(task, ctx) {
    let attempts = 0;
    let lastError = null;

    while (attempts < MAX_ATTEMPTS) {
      attempts++;

      try {
        // 1. Générer le code
        const codePrompt = PromptBuilder.generateCode(
          task,
          ctx.task.relevantFiles,
          ctx.task.relevantDecisions,
        );
        const codeResponse = await this.llm.ask(codePrompt);

        // 2. Parser et appliquer les fichiers générés
        const files = this.parseCodeResponse(codeResponse);
        await this.applyFiles(files);

        // 3. Build + validate
        const buildResult = await this.runCommand(this.techStack.build_validate);
        if (!buildResult.success) {
          lastError = { type: 'BUILD', output: buildResult.output, attempt: attempts };
          continue;
        }

        // 4. Tests
        const testResult = await this.runCommand(this.techStack.test);
        if (!testResult.success) {
          lastError = { type: 'TEST', output: testResult.output, attempt: attempts };
          continue;
        }

        // Succès — extraire les décisions prises
        const decisions = this.extractDecisions(codeResponse);
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

  // Exécuter une commande — seulement si dans allowed_commands
  async runCommand(cmd) {
    if (!this.techStack.allowed_commands?.includes(cmd) &&
        !this.techStack.allowed_commands?.some(allowed => cmd.startsWith(allowed))) {
      throw new Error(`Commande non autorisée : "${cmd}". Ajouter à tech-stack.json#allowed_commands.`);
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
  // Format attendu : ```path/to/file.js ... ``` blocs
  parseCodeResponse(response) {
    const files = [];
    const regex = /```(?:\w+)?\n?\/\/ ([^\n]+)\n([\s\S]+?)```/g;
    const pathRegex = /### ([^\n]+)\n```(?:\w+)?\n([\s\S]+?)```/g;

    let match;
    while ((match = regex.exec(response)) !== null) {
      files.push({ path: match[1].trim(), content: match[2] });
    }
    while ((match = pathRegex.exec(response)) !== null) {
      files.push({ path: match[1].trim(), content: match[2] });
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

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `runCommand` rejette toute commande non listée dans `allowed_commands` | ⬜ |
| 2 | La boucle retente exactement 3 fois avant d'escalader | ⬜ |
| 3 | `parseCodeResponse` extrait les fichiers des blocs de code | ⬜ |
| 4 | `applyFiles` crée les dossiers manquants automatiquement | ⬜ |
| 5 | `escalationContext` contient les erreurs et critères non validés | ⬜ |
| 6 | Tests unitaires mockent les commandes shell | ⬜ |
