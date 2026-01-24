# Technology Stack

## Core Technologies

### Language & Type System
- **TypeScript 5.3+**: Strict mode, full type inference
- **Target**: ES2022
- **Module**: ESM (with CJS fallback)
- **No `any` types**: Complete type safety

### UI Framework
- **Pulsar**: formular.dev's native reactive framework
- **Bundle Size**: 15KB (core)
- **Paradigm**: Signals + Effects (fine-grained reactivity)
- **Components**: Prototype-based, following formular.dev patterns

### Build System
- **Vite 5.x**: Fast dev server, optimized production builds
- **Rollup**: Library bundling with tree-shaking
- **esbuild**: Lightning-fast transpilation
- **Output**: ESM + CJS formats

### Package Manager
- **pnpm**: Workspace management, hard links
- **Workspace Protocol**: `workspace:*` for internal dependencies
- **Lockfile**: `pnpm-lock.yaml`

---

## Dependencies

### Internal Dependencies (Workspace)
```json
{
  "dependencies": {
    "formular.dev": "workspace:*",
    "@pulsar/core": "workspace:*",
    "@pulsar/design-system": "workspace:*"
  }
}
```

**Rationale**: 
- Zero external framework dependencies
- Always in sync with monorepo versions
- Can extend/modify as needed

### External Dependencies (Minimal)

#### Development Only
```json
{
  "devDependencies": {
    "typescript": "^5.3.0",
    "vite": "^5.0.0",
    "@types/node": "^20.0.0",
    "vitest": "^1.0.0",
    "playwright": "^1.40.0"
  }
}
```

#### Runtime (Zero Production Dependencies)
```json
{
  "dependencies": {}
}
```

**Rationale**:
- Pulsar, formular.dev are workspace deps (not external)
- No utility libraries (implement ourselves)
- No lodash, no date-fns, no axios
- Keep bundle minimal

---

## Styling System

### CSS Architecture
- **CSS Modules**: Scoped styles per component
- **CSS Custom Properties**: Design tokens
- **PostCSS**: Autoprefixer, nested selectors
- **No CSS-in-JS**: Keep styles in .css files

### Design Tokens (from Pulsar Design System)
```css
:root {
  /* Colors */
  --color-surface-primary: #ffffff;
  --color-surface-secondary: #f5f5f5;
  --color-border-default: #e0e0e0;
  --color-text-primary: #1a1a1a;
  --color-text-secondary: #666666;
  
  /* Spacing */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;
  
  /* Typography */
  --font-family-sans: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  --font-family-mono: "JetBrains Mono", "Fira Code", monospace;
  --font-size-xs: 11px;
  --font-size-sm: 13px;
  --font-size-md: 14px;
  --font-size-lg: 16px;
  
  /* Radius */
  --radius-sm: 2px;
  --radius-md: 4px;
  --radius-lg: 8px;
  
  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px rgba(0, 0, 0, 0.1);
  --shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.1);
}
```

---

## Testing Strategy

### Unit Testing
- **Framework**: Vitest
- **Coverage Target**: > 80%
- **Files**: `*.test.ts` alongside source files

```typescript
// Example: connection-manager.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { ConnectionManager } from './connection-manager'

describe('ConnectionManager', () => {
    let manager: IConnectionManager
    
    beforeEach(() => {
        manager = new ConnectionManager()
    })
    
    it('should connect to form instance', () => {
        const port = new MessageChannel().port1
        manager.connect('form-1', port)
        
        expect(manager.hasConnection('form-1')).toBe(true)
    })
})
```

### Integration Testing
- **Framework**: Vitest + JSDOM
- **Focus**: Message protocol, feature integration
- **Files**: `*.integration.test.ts`

### E2E Testing
- **Framework**: Playwright
- **Browsers**: Chromium, Firefox, WebKit
- **Files**: `tests/e2e/*.spec.ts`

```typescript
// Example: schema-visualizer.spec.ts
import { test, expect } from '@playwright/test'

test('should visualize form schema', async ({ page }) => {
    await page.goto('http://localhost:3000')
    
    // Wait for DevTools to connect
    await page.waitForSelector('.devtools-connected')
    
    // Click schema tab
    await page.click('[data-tab="schema"]')
    
    // Verify schema tree is rendered
    const tree = await page.$('.schema-tree')
    expect(tree).not.toBeNull()
})
```

### Visual Regression Testing
- **Framework**: Playwright (screenshots)
- **Baseline**: `tests/visual-baselines/`
- **CI**: Compare screenshots on PR

---

## Development Tools

### Code Quality
- **ESLint**: Linting with TypeScript rules
- **Prettier**: Code formatting
- **Husky**: Git hooks
- **lint-staged**: Pre-commit linting

```json
{
  "scripts": {
    "lint": "eslint src --ext .ts",
    "format": "prettier --write \"src/**/*.ts\"",
    "type-check": "tsc --noEmit"
  }
}
```

### Git Hooks
```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

pnpm lint-staged
pnpm type-check
```

### Editor Integration
- **VS Code**: Recommended extensions
  - ESLint
  - Prettier
  - TypeScript
  - Vite

```json
// .vscode/extensions.json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-typescript-next"
  ]
}
```

---

## Build Configuration

### Vite Config
```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import path from 'path'

export default defineConfig({
    build: {
        lib: {
            entry: {
                connector: path.resolve(__dirname, 'src/connector/index.ts'),
                devtools: path.resolve(__dirname, 'src/devtools/index.ts')
            },
            formats: ['es', 'cjs']
        },
        rollupOptions: {
            external: [
                'formular.dev',
                '@pulsar/core',
                '@pulsar/design-system'
            ],
            output: {
                globals: {
                    'formular.dev': 'FormularDev',
                    '@pulsar/core': 'PulsarCore',
                    '@pulsar/design-system': 'PulsarDesign'
                }
            }
        },
        sourcemap: true,
        minify: 'esbuild',
        target: 'es2022'
    },
    resolve: {
        alias: {
            '@': path.resolve(__dirname, 'src'),
            '@core': path.resolve(__dirname, 'src/core'),
            '@features': path.resolve(__dirname, 'src/features'),
            '@ui': path.resolve(__dirname, 'src/ui')
        }
    },
    test: {
        globals: true,
        environment: 'jsdom',
        coverage: {
            provider: 'v8',
            reporter: ['text', 'html', 'lcov'],
            exclude: ['**/*.test.ts', '**/*.spec.ts', '**/types/**']
        }
    }
})
```

### TypeScript Config
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@core/*": ["src/core/*"],
      "@features/*": ["src/features/*"],
      "@ui/*": ["src/ui/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

---

## CI/CD Pipeline

### GitHub Actions
```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      
      - run: pnpm install
      - run: pnpm type-check
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      
      - run: pnpm install
      - run: pnpm build
      - run: pnpm playwright install --with-deps
      - run: pnpm test:e2e
```

---

## Performance Budget

### Bundle Size Limits
```json
{
  "budgets": {
    "connector": {
      "maxSize": "5KB",
      "warning": "4.5KB"
    },
    "devtools": {
      "maxSize": "60KB",
      "warning": "55KB"
    }
  }
}
```

### Performance Targets
- **Initial Load**: < 500ms
- **State Update**: < 16ms (60fps)
- **Message Latency**: < 10ms
- **Memory Usage**: < 50MB for typical form

---

## Documentation Tools

### API Documentation
- **TypeDoc**: Generate API docs from TypeScript
- **Output**: `docs/api/`

### Example Applications
- **Vite**: Example app with DevTools
- **Storybook** (maybe): Component catalog

---

## Deployment

### NPM Package
```json
{
  "name": "formular-dev-tools",
  "version": "1.0.0",
  "main": "./dist/devtools.cjs",
  "module": "./dist/devtools.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/devtools.js",
      "require": "./dist/devtools.cjs",
      "types": "./dist/index.d.ts"
    },
    "./connector": {
      "import": "./dist/connector.js",
      "require": "./dist/connector.cjs",
      "types": "./dist/connector.d.ts"
    }
  },
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ]
}
```

### Browser Extensions
- **Chrome Web Store**: Manifest v3
- **Firefox Add-ons**: WebExtensions
- **Edge Add-ons**: Chromium-based

---

## Security Considerations

### postMessage Security
```typescript
// Validate origin
window.addEventListener('message', (event) => {
    // Only accept messages from same origin
    if (event.origin !== window.location.origin) {
        console.warn('Rejected message from', event.origin)
        return
    }
    
    // Validate message structure
    if (!isValidDevToolsMessage(event.data)) {
        console.warn('Invalid message format')
        return
    }
    
    handleMessage(event.data)
})
```

### Content Security Policy
```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; 
               script-src 'self'; 
               style-src 'self' 'unsafe-inline'; 
               connect-src 'self'">
```

### Sensitive Data Masking
```typescript
// Mask password fields
function serializeFieldValue(field: IFieldDescriptor, value: unknown): unknown {
    if (field.type === 'password') {
        return '********'
    }
    return value
}
```

---

## Summary

| Category | Technology | Rationale |
|----------|------------|-----------|
| **Language** | TypeScript 5.3+ | Type safety, modern features |
| **UI** | Pulsar | Dogfooding, zero friction |
| **Build** | Vite + Rollup | Fast dev, optimized production |
| **Package** | pnpm | Workspace management |
| **Testing** | Vitest + Playwright | Fast, comprehensive |
| **Styling** | CSS Modules + Tokens | Scoped, maintainable |
| **Linting** | ESLint + Prettier | Code quality |
| **CI/CD** | GitHub Actions | Automated testing |
| **Docs** | TypeDoc | API documentation |

**Zero external runtime dependencies** = Minimal bundle, maximum control! 🎯
