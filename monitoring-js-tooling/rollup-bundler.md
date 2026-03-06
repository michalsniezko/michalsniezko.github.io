---
layout: default
title: Rollup.js Bundler
parent: Monitoring with TICK Stack, JavaScript Tooling
nav_order: 4
---

## Rollup.js Bundler

**Use Case:** You're building a shared JS/TS library consumed by multiple frontend apps. Webpack bundles everything into one fat file (good for apps). Rollup produces clean ES modules with tree-shaking that lets the consuming app drop unused exports (good for libraries).

### Why Rollup Over Webpack for Libraries

| Concern               | Webpack                            | Rollup                              |
|-----------------------|------------------------------------|--------------------------------------|
| Output format         | Proprietary module wrapper         | Clean ES modules / CommonJS          |
| Tree-shaking          | Works, but less aggressive         | Best-in-class - dead code elimination |
| Code splitting        | Excellent for apps                 | Limited (app bundling not its strength) |
| Config complexity     | Loaders, plugins, dev server       | Minimal for library use cases        |
| Bundle size (library) | Larger due to runtime overhead     | Smaller, no runtime wrapper          |

### `rollup.config.js` for a Shared Library

```javascript
// rollup.config.js
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import typescript from '@rollup/plugin-typescript';
import { terser } from 'rollup-plugin-terser';

export default {
    input: 'src/index.ts',

    // Peer dependencies - DO NOT bundle these
    external: [
        'react',
        'react-dom',
        'axios',
        /^@company\//,  // regex: exclude all internal packages
    ],

    output: [
        {
            file: 'dist/index.cjs.js',
            format: 'cjs',           // CommonJS for Node/legacy
            sourcemap: true,
        },
        {
            file: 'dist/index.esm.js',
            format: 'esm',           // ES Modules for modern bundlers
            sourcemap: true,
        },
    ],

    plugins: [
        resolve(),                    // resolve node_modules
        commonjs(),                   // convert CJS deps to ESM
        typescript({ tsconfig: './tsconfig.build.json' }),
        terser({ format: { comments: false } }),
    ],
};
```

### `package.json` Entry Points

```json
{
    "name": "@company/ui-utils",
    "version": "2.1.0",
    "main": "dist/index.cjs.js",
    "module": "dist/index.esm.js",
    "types": "dist/index.d.ts",
    "files": ["dist"],
    "peerDependencies": {
        "react": ">=18.0.0",
        "axios": ">=1.0.0"
    },
    "scripts": {
        "build": "rollup -c",
        "build:watch": "rollup -c --watch"
    }
}
```

### Tree-Shaking in Action

```typescript
// src/index.ts - library exports 10 utilities
export { formatCurrency } from './formatters/currency';
export { formatDate } from './formatters/date';
export { debounce } from './utils/debounce';
export { throttle } from './utils/throttle';
// ... 6 more exports

// Consumer app - only imports 2
import { formatCurrency, debounce } from '@company/ui-utils';
// Rollup's ESM output lets the consumer's bundler drop the other 8 completely
```

> **Clean Code Tip:** The `external` array is the most commonly misconfigured option. If you forget to exclude `react`, Rollup bundles React into your library - every app that imports your library ships two copies of React (148KB duplicated). Rule: everything in `peerDependencies` goes in `external`. Use a regex pattern for internal packages to avoid maintaining a long list.
