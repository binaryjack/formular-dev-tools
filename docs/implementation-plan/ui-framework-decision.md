# Technology Stack Decision

## UI Framework: Pulsar

### Decision: Use Pulsar (formular.dev's native framework)

**Selected**: ✅ **Pulsar**  
**Status**: Final Decision  
**Date**: January 24, 2026

---

## Comparison Matrix

| Framework      | Bundle Size | Learning Curve             | Integration | DevTools       | Decision              |
| -------------- | ----------- | -------------------------- | ----------- | -------------- | --------------------- |
| **Pulsar**     | 15KB        | Low (same as formular.dev) | Perfect     | Need to build  | ✅ **Selected**       |
| Preact         | 3KB         | Low                        | Good        | React DevTools | ❌ Not selected       |
| React          | 45KB        | Low                        | Good        | Excellent      | ❌ Too heavy          |
| Solid.js       | 7KB         | Medium                     | Good        | Good           | ❌ Different paradigm |
| Vue            | 33KB        | Medium                     | Fair        | Excellent      | ❌ Too heavy          |
| Web Components | 0KB         | High                       | Good        | Basic          | ❌ Complex            |
| Vanilla TS     | 0KB         | N/A                        | Perfect     | None           | ❌ Too much work      |

---

## Why Pulsar?

### 1. Perfect Dogfooding Opportunity

Building DevTools with the same framework we're debugging demonstrates confidence in our technology:

```typescript
// DevTools uses formular.dev primitives
import { createSignal, createEffect } from '@pulsar/core';
import { h } from '@pulsar/core';

// Same reactive model as user's forms
const [selectedForm, setSelectedForm] = createSignal<string | null>(null);

createEffect(() => {
  const formId = selectedForm();
  if (formId) {
    // Update inspector using same reactive primitives
    updateInspector(formId);
  }
});
```

**Benefits**:

- Shows we trust our own framework
- Real-world testing of Pulsar in complex app
- Finds bugs/limitations before users do
- Marketing: "Built with what it debugs"

### 2. Zero Impedance Mismatch

DevTools speaks the same language as formular.dev:

```typescript
// User's form (formular.dev)
const form = createForm({
  schema: f.object({
    email: f.string().email(),
  }),
});

// DevTools inspector (also Pulsar)
const Inspector = function (this: IInspectorInternal, props: IInspectorProps) {
  // Same component model
  // Same reactive model
  // Same type system
};
```

**Benefits**:

- No mental model switch
- Same debugging patterns
- Consistent architecture
- Easy to understand for contributors

### 3. Bundle Size

Total DevTools bundle with Pulsar:

```
Connector:   5KB (gzipped)
├── Connection logic: 2KB
├── Message protocol: 2KB
└── Event hooks: 1KB

DevTools Window: 55KB (gzipped)
├── Pulsar Core: 15KB
├── Pulsar Components: 10KB
├── Feature modules: 25KB
└── Utilities: 5KB

Total: 60KB (vs 90KB+ with React/Vue)
```

**Comparison**:

- Pulsar: **60KB** total
- React + Libraries: **90KB+**
- Vue + Libraries: **85KB+**
- Preact + Libraries: **45KB** (but alien to formular.dev)

### 4. Consistent Architecture

Follows the same implementation rules:

```typescript
// ✅ Prototype-based (like all formular.dev code)
export const SchemaVisualizer = function(
    this: ISchemaVisualizerInternal,
    props: ISchemaVisualizerProps
) {
    Object.defineProperty(this, 'props', {
        value: props,
        writable: false,
        enumerable: false
    })
} as unknown as { new (props: ISchemaVisualizerProps): ISchemaVisualizerInternal }

// ✅ One file per item (like all formular.dev code)
schema-visualizer/
├── schema-visualizer.types.ts
├── schema-visualizer-feature.ts
├── prototype/
│   ├── render.ts
│   ├── update.ts
│   └── destroy.ts
└── index.ts

// ✅ Feature slice pattern (like all formular.dev code)
features/
├── schema-visualizer/
├── state-inspector/
└── timeline/
```

### 5. No External Dependencies

Pulsar is already part of the monorepo:

```json
{
  "dependencies": {
    "@pulsar/core": "workspace:*",
    "@pulsar/design-system": "workspace:*",
    "formular.dev": "workspace:*"
  }
}
```

**Benefits**:

- No npm install needed (workspace deps)
- Version always in sync
- Can extend Pulsar if needed
- Complete control

### 6. Reactive Primitives Match

Pulsar uses signals/effects just like formular.dev:

```typescript
// Form state (formular.dev)
const [value, setValue] = createSignal('');
const [errors, setErrors] = createSignal<string[]>([]);

createEffect(() => {
  const val = value();
  if (!val) {
    setErrors(['Required']);
  }
});

// DevTools state (Pulsar - SAME API)
const [connections, setConnections] = createSignal(new Map());
const [selected, setSelected] = createSignal(null);

createEffect(() => {
  const conn = connections().get(selected());
  if (conn) {
    updateInspector(conn);
  }
});
```

**No context switching** between debugging forms and debugging DevTools itself!

---

## Why Not Other Frameworks?

### Preact

✅ **Pros**:

- Very small (3KB)
- React-compatible
- Good DevTools

❌ **Cons**:

- Different paradigm from formular.dev
- Introduces new concepts (JSX, hooks)
- Community would need to learn React patterns
- Doesn't demonstrate confidence in Pulsar

**Verdict**: Good framework, but wrong fit for formular.dev ecosystem

### React

✅ **Pros**:

- Most popular
- Excellent DevTools
- Huge ecosystem
- Well-documented

❌ **Cons**:

- **45KB** (3x larger than Pulsar)
- Completely different from formular.dev
- Introduces alien concepts
- Doesn't dogfood our tech

**Verdict**: Overkill and sends wrong message

### Solid.js

✅ **Pros**:

- Signals-based (similar to Pulsar)
- Very fast
- Small bundle (7KB)
- Good DevTools

❌ **Cons**:

- Still different from Pulsar
- Less mature than React
- Different component model
- JSX (we don't use JSX in production)

**Verdict**: Close, but not close enough

### Web Components

✅ **Pros**:

- Standards-based
- No framework needed
- Universal compatibility
- Future-proof

❌ **Cons**:

- Complex lifecycle management
- No built-in reactivity
- Verbose syntax
- Poor debugging experience
- Would need to build everything from scratch

**Verdict**: Too much work, not worth it

### Vanilla TypeScript

✅ **Pros**:

- Zero dependencies
- Complete control
- Maximum performance

❌ **Cons**:

- Reinventing the wheel
- No reactivity system
- Tons of boilerplate
- Maintenance nightmare
- Manual DOM updates

**Verdict**: Not practical for complex UI

---

## Implementation with Pulsar

### Component Structure

```typescript
// components/schema-tree/schema-tree.types.ts
export interface ISchemaTreeProps {
  readonly schema: ISchema;
  readonly onNodeSelect?: (path: string) => void;
}

export interface ISchemaTreeInternal {
  readonly props: ISchemaTreeProps;
  readonly expanded: ISignal<Set<string>>;
  readonly selected: ISignal<string | null>;
  render(): HTMLElement;
  destroy(): void;
}

// components/schema-tree/schema-tree.ts
import { h, createSignal } from '@pulsar/core';

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

  Object.defineProperty(this, 'selected', {
    value: createSignal<string | null>(null),
    writable: false,
    enumerable: false,
  });
} as unknown as { new (props: ISchemaTreeProps): ISchemaTreeInternal };

// components/schema-tree/prototype/render.ts
export const render = function (this: ISchemaTreeInternal): HTMLElement {
  const [expanded] = this.expanded;
  const [selected, setSelected] = this.selected;

  return h(
    'div',
    { class: 'schema-tree' },
    h(
      'ul',
      { class: 'schema-tree__list' },
      this.props.schema.fields.map((field) =>
        renderNode(field, expanded(), selected(), setSelected)
      )
    )
  );
};

SchemaTree.prototype.render = render;
```

### Styling with Pulsar Design System

```typescript
// Using design tokens
import { tokens } from '@pulsar/design-system';

const styles = `
    .schema-tree {
        background: ${tokens.color.surface.primary};
        border: 1px solid ${tokens.color.border.default};
        border-radius: ${tokens.radius.md};
        padding: ${tokens.spacing.md};
    }
    
    .schema-tree__node {
        padding: ${tokens.spacing.sm};
        font-family: ${tokens.font.family.mono};
        font-size: ${tokens.font.size.sm};
    }
    
    .schema-tree__node--selected {
        background: ${tokens.color.surface.selected};
        color: ${tokens.color.text.primary};
    }
`;
```

### Reactive Updates

```typescript
// Real-time state updates from form
createEffect(() => {
  const state = formState();

  // Update inspector UI reactively
  fieldList.update(state.fields);
  validationStatus.update(state.validation);
  submissionStatus.update(state.submission);
});
```

---

## Design System Integration

### Pulsar Design System Components

Use existing Pulsar components where possible:

```typescript
import { Button, Badge, Panel, Tabs } from '@pulsar/design-system';

// DevTools tabs
const tabs = new Tabs({
  tabs: [
    { id: 'schema', label: 'Schema', icon: 'tree' },
    { id: 'state', label: 'State', icon: 'database' },
    { id: 'timeline', label: 'Timeline', icon: 'clock' },
    { id: 'performance', label: 'Performance', icon: 'chart' },
  ],
  onTabChange: (tabId) => setActiveTab(tabId),
});

// Field validation badges
const badge = new Badge({
  variant: isValid ? 'success' : 'error',
  label: isValid ? 'Valid' : 'Invalid',
});
```

### Custom DevTools Components

Build DevTools-specific components using Pulsar patterns:

```
ui/
├── devtools-specific/
│   ├── code-viewer/           # Syntax-highlighted code
│   ├── diff-viewer/           # State diff visualization
│   ├── tree-view/             # Collapsible tree
│   ├── performance-chart/     # Performance graphs
│   └── timeline/              # Event timeline
```

---

## Performance Characteristics

### Rendering Performance

```typescript
// Pulsar's fine-grained reactivity
// Only updates DOM nodes that changed

const [count, setCount] = createSignal(0);

// Only the text node updates, not entire component
h(
  'div',
  {},
  h('h1', {}, 'Static header'),
  h('p', {}, () => `Count: ${count()}`), // Only this updates
  h('button', { onClick: () => setCount(count() + 1) }, 'Increment')
);
```

**Benefits**:

- Minimal re-renders
- No virtual DOM diffing
- Direct DOM updates
- < 16ms frame time

### Memory Usage

- Signal subscriptions: O(n) where n = reactive values
- Component tree: O(m) where m = components
- No VDOM overhead
- Efficient cleanup on destroy

### Bundle Splitting

```typescript
// vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'pulsar-core': ['@pulsar/core'],
          'pulsar-ui': ['@pulsar/design-system'],
          'schema-visualizer': ['./src/features/schema-visualizer'],
          'state-inspector': ['./src/features/state-inspector'],
        },
      },
    },
  },
};
```

Lazy-load features on demand:

```typescript
// Load schema visualizer only when tab is opened
const loadSchemaVisualizer = async () => {
  const { SchemaVisualizerFeature } = await import('./features/schema-visualizer');
  return new SchemaVisualizerFeature();
};
```

---

## Developer Experience

### Same Patterns Everywhere

Developers familiar with formular.dev will instantly understand DevTools code:

✅ Same file structure  
✅ Same naming conventions  
✅ Same prototype patterns  
✅ Same reactive primitives  
✅ Same type system

### Easy Contributions

Lower barrier to entry for contributors:

1. Already know Pulsar from using formular.dev
2. No need to learn React/Vue/etc.
3. Can jump between formular.dev and DevTools easily
4. Same code review standards apply

---

## Final Decision Summary

**Framework**: Pulsar  
**Why**: Dogfooding, consistency, zero friction, perfect fit  
**Bundle**: 60KB total (competitive)  
**DX**: Seamless for formular.dev developers  
**Future**: Can evolve Pulsar based on DevTools learnings

**Alternative Considered**: Preact (rejected - different paradigm)  
**Status**: ✅ Final, no further discussion needed

---

## Next Steps

1. ✅ Set up Pulsar in formular-dev-tools package
2. ✅ Create base layout components
3. ✅ Implement connection protocol
4. ✅ Build first feature (Schema Visualizer)
5. ✅ Iterate and refine

**Let's build DevTools with Pulsar!** 🚀
