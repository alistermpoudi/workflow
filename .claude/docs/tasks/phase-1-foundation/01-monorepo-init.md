# Phase 1 — Tâche 1.1 : Initialisation du Projet

## Objectif

Créer la structure complète du projet avec les bons outils de développement : `package.json`, structure `src/`, configuration ESLint, Vitest pour les tests, et les fichiers de config nécessaires.

## Fichiers à Créer

```
workflow/
├── package.json
├── eslint.config.js
├── vitest.config.js
├── .nvmrc
├── src/
│   ├── core/           (dossiers vides avec .gitkeep)
│   ├── phases/
│   ├── tools/
│   ├── interfaces/
│   └── llm/
└── tests/
    └── unit/
        └── .gitkeep
```

## Implémentation

### `package.json`

```json
{
  "name": "workflow",
  "version": "0.1.0",
  "description": "Agent de code avec mémoire projet persistante",
  "type": "module",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "node --watch src/index.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src/ tests/",
    "build:validate": "npm run lint && npm test"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.30.0",
    "@modelcontextprotocol/sdk": "^1.0.0",
    "chalk": "^5.3.0",
    "zod": "^3.22.0"
  },
  "devDependencies": {
    "@antfu/eslint-config": "^3.0.0",
    "eslint": "^9.0.0",
    "vitest": "^2.0.0"
  },
  "engines": {
    "node": ">=22.0.0"
  }
}
```

### `vitest.config.js`

```javascript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',
    include: ['tests/**/*.test.js'],
    coverage: {
      reporter: ['text', 'lcov'],
    },
  },
});
```

### `eslint.config.js`

```javascript
import antfu from '@antfu/eslint-config';

export default antfu({
  type: 'lib',
  typescript: false,
  stylistic: {
    indent: 2,
    quotes: 'single',
    semi: true,
  },
});
```

### `.nvmrc`

```
22
```

## Critères de Validation

| # | Critère | Vérifié |
|---|---------|---------|
| 1 | `npm install` s'exécute sans erreur | ⬜ |
| 2 | `npm run lint` passe sur les dossiers src/ et tests/ vides | ⬜ |
| 3 | `npm test` s'exécute (0 tests = OK pour cette tâche) | ⬜ |
| 4 | Structure de dossiers correcte | ⬜ |
