# Architecture Overview

## Core Design Principles

### 1. Protocol-Based Communication

DevTools operates as a **separate window** that communicates with form instances through a well-defined protocol.

```
┌─────────────────────┐           postMessage            ┌─────────────────────┐
│   User Application  │◄────────────────────────────────►│  DevTools Window    │
│                     │                                   │                     │
│  ┌───────────────┐  │                                   │  ┌───────────────┐  │
│  │ formular.dev  │  │                                   │  │  Visualizers  │  │
│  │    Forms      │  │                                   │  │   Inspectors  │  │
│  └───────┬───────┘  │                                   │  └───────┬───────┘  │
│          │          │                                   │          │          │
│  ┌───────▼───────┐  │                                   │  ┌───────▼───────┐  │
│  │ DevTools      │  │                                   │  │   DevTools    │  │
│  │ Connector     │  │                                   │  │   Core        │  │
│  └───────────────┘  │                                   │  └───────────────┘  │
└─────────────────────┘                                   └─────────────────────┘
```

### 2. Layered Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    UI Layer (Pulsar)                        │
│  - Schema Visualizer    - Timeline       - Profiler         │
│  - State Inspector      - Performance    - Playground       │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                   Feature Layer                              │
│  - SchemaVisualizerFeature                                  │
│  - StateInspectorFeature                                    │
│  - TimelineFeature                                          │
│  - PerformanceFeature                                       │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    Core Layer                               │
│  - Connection Manager    - State Manager                    │
│  - Message Protocol      - Event Bus                        │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                  Connector (User App)                       │
│  - Form Inspector       - Event Interceptor                 │
│  - State Serializer     - Message Sender                    │
└─────────────────────────────────────────────────────────────┘
```

## Component Structure

Following formular.dev implementation rules:

### Core Components

#### 1. Connection System

```
core/
├── connection/
│   ├── connection.types.ts
│   ├── connection-manager.ts
│   ├── prototype/
│   │   ├── connect.ts
│   │   ├── disconnect.ts
│   │   └── send-message.ts
│   └── index.ts
├── messaging/
│   ├── message.types.ts
│   ├── message-protocol.ts
│   ├── protocol/
│   │   ├── serialize-state.ts
│   │   ├── deserialize-state.ts
│   │   └── validate-message.ts
│   └── index.ts
└── state/
    ├── state.types.ts
    ├── devtools-state.ts
    ├── prototype/
    │   ├── update-state.ts
    │   ├── get-snapshot.ts
    │   └── subscribe.ts
    └── index.ts
```

#### 2. Feature Structure

Each feature follows the same pattern:

```
features/
├── schema-visualizer/
│   ├── schema-visualizer.types.ts
│   ├── schema-visualizer-feature.ts
│   ├── components/
│   │   ├── schema-tree.ts
│   │   ├── schema-node.ts
│   │   └── type-badge.ts
│   ├── logic/
│   │   ├── parse-schema.ts
│   │   ├── flatten-tree.ts
│   │   └── infer-types.ts
│   └── index.ts
```

## Prototype-Based Architecture

All classes use function constructors:

```typescript
// connection/connection-manager.ts
export const ConnectionManager = function (this: IConnectionManagerInternal) {
  Object.defineProperty(this, 'connections', {
    value: new Map<string, IConnection>(),
    writable: false,
    enumerable: false,
  });

  Object.defineProperty(this, 'eventBus', {
    value: createEventBus(),
    writable: false,
    enumerable: false,
  });
} as unknown as { new (): IConnectionManagerInternal };

// Prototype methods in separate files
export const connect = function (
  this: IConnectionManagerInternal,
  formId: string,
  port: MessagePort
): void {
  // Implementation
};

ConnectionManager.prototype.connect = connect;
```

## Message Protocol

### Message Types

```typescript
// messaging/message.types.ts
export type DevToolsMessageType =
  | 'CONNECT'
  | 'DISCONNECT'
  | 'STATE_UPDATE'
  | 'VALIDATE'
  | 'SUBMIT'
  | 'FIELD_CHANGE'
  | 'ERROR'
  | 'PERFORMANCE_SAMPLE';

export interface IDevToolsMessage<T = unknown> {
  readonly type: DevToolsMessageType;
  readonly formId: string;
  readonly timestamp: number;
  readonly payload: T;
}

export interface IStateUpdatePayload {
  readonly fields: Record<string, IFieldState>;
  readonly validationState: IValidationState;
  readonly submissionState: ISubmissionState;
}

export interface IPerformanceSample {
  readonly operation: 'render' | 'validate' | 'submit';
  readonly duration: number;
  readonly timestamp: number;
}
```

### Protocol Flow

```typescript
// User App → DevTools
{
    type: 'STATE_UPDATE',
    formId: 'login-form',
    timestamp: 1706112345678,
    payload: {
        fields: {
            email: { value: 'user@example.com', valid: true },
            password: { value: '********', valid: true }
        },
        validationState: { isValid: true, errors: [] },
        submissionState: { isSubmitting: false }
    }
}

// DevTools → User App (validation request)
{
    type: 'VALIDATE',
    formId: 'login-form',
    timestamp: 1706112345890,
    payload: { fieldName: 'email' }
}
```

## State Management

### DevTools State Structure

```typescript
export interface IDevToolsState {
  readonly connections: Map<string, IFormConnection>;
  readonly selectedFormId: string | null;
  readonly activeTab: TabId;
  readonly settings: IDevToolsSettings;
}

export interface IFormConnection {
  readonly formId: string;
  readonly name: string;
  readonly state: IFormState;
  readonly history: IStateSnapshot[];
  readonly performance: IPerformanceMetrics;
}

export interface IStateSnapshot {
  readonly timestamp: number;
  readonly state: IFormState;
  readonly trigger: string;
}
```

### Reactive State

Uses formular.dev's reactive primitives:

```typescript
import { createSignal, createEffect } from '@pulsar/core';

// DevTools state is reactive
const [connections, setConnections] = createSignal<Map<string, IFormConnection>>(new Map());
const [selectedFormId, setSelectedFormId] = createSignal<string | null>(null);

// Auto-update UI when state changes
createEffect(() => {
  const formId = selectedFormId();
  if (formId) {
    updateInspector(connections().get(formId));
  }
});
```

## UI Component Architecture

### Pulsar Component Pattern

```typescript
// components/schema-tree.ts
import { h } from '@pulsar/core';

export const SchemaTree = function (this: ISchemaTreeInternal, props: ISchemaTreeProps) {
  Object.defineProperty(this, 'props', {
    value: props,
    writable: false,
    enumerable: false,
  });

  Object.defineProperty(this, 'expanded', {
    value: createSignal(new Set<string>()),
    writable: false,
    enumerable: false,
  });
} as unknown as { new (props: ISchemaTreeProps): ISchemaTreeInternal };

// Render method
export const render = function (this: ISchemaTreeInternal): HTMLElement {
  const [expanded, setExpanded] = this.expanded;

  return h(
    'div',
    { class: 'schema-tree' },
    this.props.schema.fields.map((field) =>
      SchemaNode({
        field,
        expanded: expanded().has(field.name),
        onToggle: () => toggleExpanded(field.name),
      })
    )
  );
};

SchemaTree.prototype.render = render;
```

## Extension Points

### Plugin System (Phase 3)

```typescript
export interface IDevToolsPlugin {
  readonly name: string;
  readonly version: string;
  initialize(devtools: IDevTools): void;
  onFormConnect?(formId: string): void;
  onStateUpdate?(formId: string, state: IFormState): void;
}

// Register custom plugins
devtools.registerPlugin(customValidatorPlugin);
```

## Performance Considerations

1. **Lazy Loading**: Features load on-demand
2. **Virtual Scrolling**: For large form lists and timelines
3. **Debounced Updates**: State updates batched every 16ms
4. **Memoization**: Schema tree nodes memoized by path
5. **Message Throttling**: Performance samples rate-limited

## Testing Strategy

```
tests/
├── unit/
│   ├── connection/
│   ├── messaging/
│   └── state/
├── integration/
│   ├── features/
│   └── protocol/
└── e2e/
    ├── scenarios/
    └── fixtures/
```

## Security Considerations

1. **Origin Validation**: Verify postMessage origin
2. **Message Sanitization**: Validate all incoming messages
3. **Sensitive Data**: Option to mask password fields
4. **Production Mode**: DevTools disabled by default in production
5. **CSP Compliance**: Works with Content Security Policy

## Deployment Options

### Option 1: Standalone Window (Phase 1)

```typescript
import { openDevTools } from 'formular-dev-tools';

if (import.meta.env.DEV) {
  openDevTools({ port: 3001 });
}
```

### Option 2: Browser Extension (Phase 3)

- Chrome DevTools panel integration
- Firefox DevTools integration
- Edge DevTools integration

### Option 3: Embedded Panel (Future)

```typescript
import { EmbeddedDevTools } from 'formular-dev-tools/embedded'

<EmbeddedDevTools formId="login-form" position="bottom-right" />
```

## Build System

### Vite Configuration

```typescript
// vite.config.ts
export default defineConfig({
  build: {
    lib: {
      entry: {
        connector: './src/connector/index.ts',
        devtools: './src/devtools/index.ts',
      },
      formats: ['es', 'cjs'],
    },
    rollupOptions: {
      external: ['formular.dev', '@pulsar/core'],
    },
  },
});
```

### Output

- `connector.js`: Embedded in user app (5KB gzipped)
- `devtools.js`: DevTools window (50KB gzipped)
- Tree-shakable: Only used features included

## Integration with Monorepo

```yaml
# pnpm-workspace.yaml (already exists)
packages:
  - 'packages/*'

# packages/formular-dev-tools/package.json
{
  "name": "formular-dev-tools",
  "dependencies": {
    "formular.dev": "workspace:*",
    "@pulsar/core": "workspace:*",
    "@pulsar/design-system": "workspace:*"
  }
}
```

Perfect dogfooding: DevTools uses the same libraries it debugs!
