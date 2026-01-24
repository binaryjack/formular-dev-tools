# Formular DevTools

> Developer Tools for formular.dev - Visualize, debug, and optimize your forms

## Overview

Formular DevTools is a comprehensive developer experience tool for [formular.dev](https://github.com/binaryjack/formular.dev), providing visual inspection, debugging, and optimization capabilities for form development.

## Features

### Phase 1: Core Inspection (MVP)

- **Schema Visualizer**: Interactive tree view of form schemas with type inference
- **State Inspector**: Real-time form state, field values, and validation status
- **Timeline**: Chronological log of form events (mount, validation, submission)
- **Performance Monitor**: Track render cycles, validation timing, and submission latency

### Phase 2: Advanced Debugging

- **Country Validator Playground**: Test phone/postal/AHV validators with sample data
- **i18n Error Preview**: Switch between languages to preview error messages
- **Submission Strategy Debugger**: Visualize submission flow (Direct vs Context strategies)
- **Error Boundary Tracker**: Monitor error boundary activations and recovery

### Phase 3: Optimization Tools

- **IoC Container Inspector**: View dependency injection container state and graph
- **Time Travel Debugger**: Record and replay form state changes
- **Validation Profiler**: Identify slow validation rules
- **Bundle Analyzer**: Show which formular.dev modules are being used

## Unique Value Propositions

Unlike other form library DevTools (TanStack, React Hook Form):

- **Country Validators**: Dedicated UI for testing region-specific validators (12 countries)
- **i18n-First**: Built-in multi-language error preview (6 languages supported)
- **IoC/DI Insights**: Visualize dependency injection container (unique to formular.dev)
- **Strategy Pattern Debugger**: Understand submission flow through different strategies
- **Schema-First Design**: Visual representation of Zod-inspired schema definitions

## Architecture

Built using the same architectural patterns as the formular.dev ecosystem:

- **Prototype-based Classes**: No ES6 classes, function constructors only
- **Feature Slice Pattern**: Organized by feature/domain
- **Zero Dependencies**: Minimal external dependencies
- **Type-Safe**: Full TypeScript, no `any` types
- **Reactive Core**: Uses formular.dev's own reactive system

## Technology Stack

### UI Framework

**Pulsar** (formular.dev's native UI framework)

- Built on the same reactive primitives as formular.dev
- Prototype-based component model
- Zero framework overhead
- Perfect dogfooding opportunity

### Styling

- **Pulsar Design System**: Consistent with formular.dev branding
- CSS custom properties for theming
- Responsive grid system

### Build System

- **Vite**: Fast development and optimized production builds
- **TypeScript**: Strict mode, full type safety
- **pnpm**: Workspace integration with formular.dev monorepo

### DevTools Integration

- **Standalone Window**: Separate window connected via postMessage
- **Browser Extension** (future): Chrome/Firefox extension for in-page inspection

## Getting Started

```bash
# Install dependencies
pnpm install

# Development mode
pnpm dev

# Build for production
pnpm build

# Run tests
pnpm test
```

## Integration with formular.dev

Add DevTools to your form application:

```typescript
import { createForm } from 'formular.dev';
import { connectDevTools } from 'formular-dev-tools';

const form = createForm({
  schema: f.object({
    email: f.string().email(),
    password: f.string().min(8),
  }),
});

// Connect to DevTools (development only)
if (import.meta.env.DEV) {
  connectDevTools(form, {
    name: 'Login Form',
    position: 'bottom-right',
  });
}
```

## Project Structure

```
formular-dev-tools/
├── src/
│   ├── core/               # Core DevTools infrastructure
│   │   ├── connection/     # Form instance connection
│   │   ├── messaging/      # postMessage protocol
│   │   └── state/          # DevTools state management
│   ├── features/           # Feature-specific UI
│   │   ├── schema-visualizer/
│   │   ├── state-inspector/
│   │   ├── timeline/
│   │   └── performance/
│   ├── ui/                 # Shared UI components
│   │   ├── layout/
│   │   ├── panels/
│   │   └── primitives/
│   └── index.ts
├── docs/
│   ├── implementation-plan/
│   │   ├── architecture.md
│   │   ├── phases.md
│   │   ├── technology-stack.md
│   │   └── ui-framework-decision.md
│   └── api/
├── tests/
└── package.json
```

## Development Principles

Following formular.dev's implementation rules:

1. **One Item Per File**: Each enum, interface, const in separate files
2. **Prototype Pattern**: Function constructors with prototype methods
3. **Type Safety**: No `any` types, proper interfaces everywhere
4. **Feature Slice**: Organize by feature, not layer
5. **Kebab-case Naming**: All files use kebab-case

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for development guidelines.

## License

MIT © [BinaryJack](https://github.com/binaryjack)

---

**Status**: 🚧 In Development - Phase 1 (MVP)
