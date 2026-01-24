# Formular DevTools - Implementation Rules

> **Critical**: These rules are **mandatory** for all code in formular-dev-tools. They ensure consistency with the formular.dev ecosystem.

---

## Core Architectural Patterns

### 0. File Naming: kebab-case ALWAYS

```
✅ CORRECT:
schema-visualizer.ts
connection-manager.ts
devtools-state.types.ts
schema-tree.component.ts

❌ WRONG:
SchemaVisualizer.ts
connectionManager.ts
DevToolsState.types.ts
schemaTree.component.ts
```

---

### 1. One Item Per File

**Rules**:
- One enum per file: `[feature].enum.ts`
- One interface per file: `[feature].interface.ts`
- One const per file: `[feature].ts`
- Components can have their props interface in the same file
- Prototype methods ALWAYS in `prototype/` folder: `[method-name].ts`

**Example**:
```
connection/
├── connection.types.ts              # Public interface
├── connection-manager.ts            # Constructor
├── connection-manager.types.ts      # Internal interface
├── message-type.enum.ts             # Message type enum
├── connection-state.enum.ts         # Connection state enum
├── prototype/
│   ├── connect.ts                   # Connect method
│   ├── disconnect.ts                # Disconnect method
│   ├── send-message.ts              # Send message method
│   └── has-connection.ts            # Has connection method
└── index.ts                         # Public exports
```

---

### 2. Feature Slice Pattern

Organize by **feature/domain**, NOT by technical layer.

```
src/
├── core/
│   ├── connection/                  # Feature: Connection System
│   │   ├── connection.types.ts
│   │   ├── connection-manager.ts
│   │   ├── prototype/
│   │   └── index.ts
│   ├── messaging/                   # Feature: Message Protocol
│   │   ├── message.types.ts
│   │   ├── message-protocol.ts
│   │   └── index.ts
│   └── state/                       # Feature: State Management
│       ├── state.types.ts
│       ├── devtools-state.ts
│       └── index.ts
├── features/
│   ├── schema-visualizer/           # Feature: Schema Visualizer
│   │   ├── schema-visualizer.types.ts
│   │   ├── schema-visualizer-feature.ts
│   │   ├── components/
│   │   │   ├── schema-tree.ts
│   │   │   ├── schema-node.ts
│   │   │   └── type-badge.ts
│   │   ├── logic/
│   │   │   ├── parse-schema.ts
│   │   │   └── flatten-tree.ts
│   │   └── index.ts
│   ├── state-inspector/             # Feature: State Inspector
│   └── timeline/                    # Feature: Timeline
└── ui/
    ├── layout/
    └── primitives/
```

**Benefits**:
- Clear feature boundaries
- Easy to find related code
- Features can be understood independently
- Simple to add/remove features

---

### 3. Prototype-Based Classes Only

**NO ES6 classes. NO `class` keyword. Use function constructors ONLY.**

#### ❌ WRONG - ES6 Class:
```typescript
class ConnectionManager {
    private connections: Map<string, IConnection>
    
    constructor() {
        this.connections = new Map()
    }
    
    connect(formId: string, port: MessagePort): void {
        this.connections.set(formId, { formId, port })
    }
}
```

#### ✅ CORRECT - Prototype Pattern:
```typescript
// connection-manager.ts
export const ConnectionManager = function(
    this: IConnectionManagerInternal
) {
    Object.defineProperty(this, 'connections', {
        value: new Map<string, IConnection>(),
        writable: false,
        enumerable: false
    })
} as unknown as { new (): IConnectionManagerInternal }

// prototype/connect.ts
export const connect = function(
    this: IConnectionManagerInternal,
    formId: string,
    port: MessagePort
): void {
    const connection: IConnection = { formId, port }
    this.connections.set(formId, connection)
}

// Attach to prototype
ConnectionManager.prototype.connect = connect
```

**Key Requirements**:
1. Constructor is a function, not a class
2. Methods are separate functions assigned to prototype
3. Use `Object.defineProperty` for private/internal properties
4. Type cast constructor with `as unknown as { new ... }`
5. Each method in its own file under `prototype/` directory

---

### 4. Type Safety - No `any`, No Stub Types

**Always create proper interfaces and types. NEVER use `any` or placeholder types.**

#### ❌ WRONG:
```typescript
// Using 'any'
function process(data: any) {
    return data.value
}

// Stub typing
function transform(input: unknown): unknown {
    // TODO: type this later
    return input
}

// Missing types (TypeScript infers 'any')
export const analyze = function(node) {
    return node.children.map(child => {
        // TypeScript infers 'any'
    })
}
```

#### ✅ CORRECT:
```typescript
// Proper interface
interface IMessagePayload {
    readonly type: string
    readonly data: IFormState
    readonly timestamp: number
}

function processMessage(payload: IMessagePayload): IFormState {
    return payload.data
}

// Union type for variants
type ConnectionState = 'connected' | 'disconnected' | 'connecting' | 'error'

function updateState(state: ConnectionState): void {
    if (state === 'error') {
        // Type-safe handling
    }
}

// Generic with constraints
interface IFeature<T extends IFeatureConfig> {
    initialize(config: T): void
    destroy(): void
}

export const Feature = function<T extends IFeatureConfig>(
    this: IFeatureInternal<T>,
    config: T
): void {
    // Properly typed
}
```

**Type Organization**:
```typescript
// feature/feature.types.ts - Public interfaces
export interface ISchemaVisualizer {
    render(): HTMLElement
    update(schema: ISchema): void
    destroy(): void
}

// Internal interface with private details
export interface ISchemaVisualizerInternal extends ISchemaVisualizer {
    readonly props: ISchemaVisualizerProps
    readonly expanded: ISignal<Set<string>>
    handleNodeClick(path: string): void
}

// Type aliases for complex unions
export type SchemaNode = IStringSchema | INumberSchema | IObjectSchema | IArraySchema

// Enum-like types
export type PanelPosition = 'left' | 'right' | 'bottom' | 'floating'
```

**Type Guards**:
```typescript
// Type guard for union discrimination
export function isConnection(
    value: unknown
): value is IConnection {
    return (
        typeof value === 'object' &&
        value !== null &&
        'formId' in value &&
        'port' in value
    )
}

// Usage with type narrowing
if (isConnection(data)) {
    // TypeScript knows this is IConnection
    console.log(data.formId)
}
```

---

### 5. Factory Functions for Instantiation

Provide factory functions alongside constructors for ergonomic API.

```typescript
// Constructor (for internal use)
export const ConnectionManager = function(
    this: IConnectionManagerInternal
) {
    Object.defineProperty(this, 'connections', {
        value: new Map<string, IConnection>(),
        writable: false,
        enumerable: false
    })
} as unknown as { new (): IConnectionManagerInternal }

// Factory function (public API)
export function createConnectionManager(): IConnectionManager {
    const manager = new ConnectionManager() as unknown as IConnectionManagerInternal
    
    // Bind methods
    manager.connect = connect.bind(manager)
    manager.disconnect = disconnect.bind(manager)
    manager.hasConnection = hasConnection.bind(manager)
    
    // Return as public interface (hides internals)
    return manager as unknown as IConnectionManager
}
```

---

### 6. Immutable API Surface

Public interfaces should be `readonly` where appropriate.

```typescript
export interface IDevToolsConfig {
    readonly port: number
    readonly position: PanelPosition
    readonly theme: 'light' | 'dark' | 'auto'
    
    // Mutable only through methods
    onFormConnect?: (formId: string) => void
}

export interface IFormState {
    readonly formId: string
    readonly fields: ReadonlyMap<string, IFieldState>
    readonly validationState: Readonly<IValidationState>
    readonly timestamp: number
}
```

---

### 7. Context Objects for Shared State

Pass context objects through pipelines instead of global state.

```typescript
// Context definition
export interface IDevToolsContext {
    readonly connectionManager: IConnectionManager
    readonly stateManager: IStateManager
    readonly eventBus: IEventBus
    readonly config: IDevToolsConfig
}

// Constructor accepts context
export const SchemaVisualizerFeature = function(
    this: ISchemaVisualizerFeatureInternal,
    context: IDevToolsContext
) {
    Object.defineProperty(this, 'context', {
        value: context,
        writable: false,
        enumerable: false
    })
} as unknown as { new (context: IDevToolsContext): ISchemaVisualizerFeatureInternal }
```

---

### 8. Reactive Updates with Pulsar Primitives

Use Pulsar's signals and effects for reactivity.

```typescript
import { createSignal, createEffect } from '@pulsar/core'

// Reactive state
const [connections, setConnections] = createSignal<Map<string, IConnection>>(new Map())
const [selectedFormId, setSelectedFormId] = createSignal<string | null>(null)

// Auto-update UI when state changes
createEffect(() => {
    const formId = selectedFormId()
    const conn = connections().get(formId)
    
    if (conn && formId) {
        updateInspectorUI(conn)
    }
})
```

---

## Component Patterns

### Pulsar Component Structure

```typescript
// components/schema-tree/schema-tree.types.ts
export interface ISchemaTreeProps {
    readonly schema: ISchema
    readonly onNodeSelect?: (path: string) => void
}

export interface ISchemaTreeInternal {
    readonly props: ISchemaTreeProps
    readonly expanded: ISignal<Set<string>>
    readonly selected: ISignal<string | null>
    render(): HTMLElement
    destroy(): void
}

// components/schema-tree/schema-tree.ts
import { h, createSignal } from '@pulsar/core'

export const SchemaTree = function(
    this: ISchemaTreeInternal,
    props: ISchemaTreeProps
) {
    Object.defineProperty(this, 'props', {
        value: props,
        writable: false,
        enumerable: false
    })
    
    Object.defineProperty(this, 'expanded', {
        value: createSignal(new Set<string>()),
        writable: false,
        enumerable: false
    })
    
    Object.defineProperty(this, 'selected', {
        value: createSignal<string | null>(null),
        writable: false,
        enumerable: false
    })
} as unknown as { new (props: ISchemaTreeProps): ISchemaTreeInternal }

// components/schema-tree/prototype/render.ts
export const render = function(this: ISchemaTreeInternal): HTMLElement {
    const [expanded] = this.expanded
    const [selected, setSelected] = this.selected
    
    return h('div', { class: 'schema-tree' },
        h('ul', { class: 'schema-tree__list' },
            this.props.schema.fields.map(field => 
                renderNode(field, expanded(), selected(), setSelected)
            )
        )
    )
}

SchemaTree.prototype.render = render

// components/schema-tree/prototype/destroy.ts
export const destroy = function(this: ISchemaTreeInternal): void {
    // Cleanup subscriptions
    const [, , unsubscribe] = this.expanded
    unsubscribe()
}

SchemaTree.prototype.destroy = destroy
```

---

## Messaging Protocol Patterns

### Message Type Definitions

```typescript
// messaging/message-type.enum.ts
export enum MessageType {
    CONNECT = 'CONNECT',
    DISCONNECT = 'DISCONNECT',
    STATE_UPDATE = 'STATE_UPDATE',
    VALIDATE = 'VALIDATE',
    SUBMIT = 'SUBMIT',
    FIELD_CHANGE = 'FIELD_CHANGE',
    ERROR = 'ERROR',
    PERFORMANCE_SAMPLE = 'PERFORMANCE_SAMPLE'
}

// messaging/message.types.ts
export interface IDevToolsMessage<T = unknown> {
    readonly type: MessageType
    readonly formId: string
    readonly timestamp: number
    readonly payload: T
}

export interface IStateUpdatePayload {
    readonly fields: Record<string, IFieldState>
    readonly validationState: IValidationState
    readonly submissionState: ISubmissionState
}
```

### Message Validation

```typescript
// messaging/protocol/validate-message.ts
export function isValidMessage(data: unknown): data is IDevToolsMessage {
    if (typeof data !== 'object' || data === null) {
        return false
    }
    
    const msg = data as Record<string, unknown>
    
    return (
        typeof msg.type === 'string' &&
        Object.values(MessageType).includes(msg.type as MessageType) &&
        typeof msg.formId === 'string' &&
        typeof msg.timestamp === 'number' &&
        'payload' in msg
    )
}
```

---

## Testing Patterns

### Type-Safe Test Fixtures

```typescript
// test/fixtures/connection.fixture.ts
export interface IConnectionTestFixture {
    manager: IConnectionManager
    formId: string
    port: MessagePort
}

export function createConnectionFixture(): IConnectionTestFixture {
    const channel = new MessageChannel()
    
    return {
        manager: createConnectionManager(),
        formId: 'test-form-1',
        port: channel.port1
    }
}

// test/connection/connect.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { createConnectionFixture } from '../fixtures/connection.fixture'

describe('ConnectionManager.connect', () => {
    let fixture: IConnectionTestFixture
    
    beforeEach(() => {
        fixture = createConnectionFixture()
    })
    
    it('should establish connection', () => {
        fixture.manager.connect(fixture.formId, fixture.port)
        
        expect(fixture.manager.hasConnection(fixture.formId)).toBe(true)
    })
})
```

---

## Naming Conventions

### Files
- **kebab-case**: `schema-visualizer.ts`
- **Types**: `*.types.ts`
- **Tests**: `*.test.ts`
- **Enums**: `*.enum.ts`
- **Constants**: `*.const.ts`

### Interfaces
- **Public**: `ISchemaVisualizer`
- **Internal**: `ISchemaVisualizerInternal`
- **Props**: `ISchemaVisualizerProps`
- **Config**: `ISchemaVisualizerConfig`

### Functions
- **Factory**: `createSchemaVisualizer()`
- **Predicate**: `isValidMessage()`
- **Handler**: `handleNodeClick()`

### Constants
- **UPPER_SNAKE_CASE**: `MAX_CONNECTIONS`
- **Enums**: PascalCase members

---

## Module Exports

### Index Files

```typescript
// feature/index.ts
export { Feature } from './feature'
export { createFeature } from './create-feature'
export type { IFeature, IFeatureConfig } from './feature.types'

// DO NOT export internal types
// DO NOT export prototype methods directly
```

---

## Summary Checklist

Before committing code, verify:

- [ ] File names are kebab-case
- [ ] One item per file (enum, interface, const)
- [ ] Prototype methods in `prototype/` folder
- [ ] No ES6 classes, only function constructors
- [ ] No `any` types, proper interfaces everywhere
- [ ] Factory functions for public API
- [ ] `readonly` for immutable properties
- [ ] Context objects passed, not globals
- [ ] Pulsar signals/effects for reactivity
- [ ] Type guards for union discrimination
- [ ] Tests for all features
- [ ] Public exports in index files only

---

**These rules are non-negotiable. They ensure formular-dev-tools maintains the same high quality and consistency as the rest of the formular.dev ecosystem.** 🎯
