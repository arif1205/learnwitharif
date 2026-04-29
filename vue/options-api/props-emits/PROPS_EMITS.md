# Props & Emits — Options API

---

## How Props Work

Props flow down — parent owns the data, child receives a read-only copy. Vue warns if you mutate a prop. For two-way binding, use emits + `v-model`. Under the hood, props are reactive — child re-renders when parent's bound value changes.

---

## Declaring Props

```js
import type { PropType } from 'vue'

export default {
  props: {
    // basic types
    card:     { type: Object as PropType<Card>, required: true },
    readonly: { type: Boolean, default: false },
    maxCards: { type: Number, default: 10 },
    title:    { type: String, required: true },

    // union types
    priority: {
      type: String as PropType<'low' | 'medium' | 'high'>,
      default: 'medium'
    },

    // array
    cards: {
      type: Array as PropType<Card[]>,
      default: (): Card[] => []   // factory function for object/array defaults
    },

    // validator
    status: {
      type: String,
      validator: (v: string) => ['todo', 'in-progress', 'done'].includes(v)
    }
  }
}
```

**Object/array defaults must be factory functions.** If you write `default: []`, all component instances share the same array reference.

```js
// BAD — all KanbanColumn instances share the same cards array
cards: { type: Array, default: [] }

// GOOD
cards: { type: Array as PropType<Card[]>, default: (): Card[] => [] }
```

---

## Using Props

```js
export default {
  props: { card: { type: Object as PropType<Card>, required: true } },

  computed: {
    priorityClass(): string {
      return `card--${this.card.priority}`   // this.propName
    }
  },

  methods: {
    logCard() {
      console.log(this.card.title, this.card.priority)
    }
  }
}
```

**Never mutate props:**
```js
methods: {
  updateTitle(v: string) {
    this.card.title = v         // BAD — mutating prop object
    this.$emit('update', v)     // GOOD — emit to parent
  }
}
```

---

## `$attrs` — Non-Prop Attributes

Attributes passed by parent that aren't declared in `props` or `emits` fall through to the root element automatically. Useful for forwarding HTML attributes.

```html
<!-- Parent -->
<KanbanCard data-testid="card-1" class="highlighted" :card="card" />
```

```js
// KanbanCard.vue — data-testid and class go to root <div> automatically
export default {
  props: ['card'],
  inheritAttrs: false   // disable auto-fallthrough
}
```

```html
<!-- template — manually place $attrs -->
<div class="card-wrapper">
  <div class="card" v-bind="$attrs">  <!-- forward to inner element -->
    {{ card.title }}
  </div>
</div>
```

---

## Declaring Emits

Declaring emits is not optional. Vue uses the list to distinguish component events from native DOM events, which prevents duplicate event firing.

```js
export default {
  emits: ['move', 'delete', 'update:title'],

  // with validation
  emits: {
    move:   (columnId: number) => typeof columnId === 'number',
    delete: (cardId: number)   => cardId > 0,
    'update:title': (title: string) => title.trim().length > 0
  }
}
```

---

## Emitting Events

```js
export default {
  props: { card: { type: Object as PropType<Card>, required: true } },
  emits: ['move', 'delete', 'update:title'],

  methods: {
    moveToColumn(columnId: number) {
      this.$emit('move', columnId)
    },

    deleteCard() {
      this.$emit('delete', this.card.id)
    },

    saveTitle(newTitle: string) {
      this.$emit('update:title', newTitle)
    }
  }
}
```

```html
<!-- Parent -->
<KanbanCard
  :card="card"
  @move="handleMove"
  @delete="handleDelete"
  @update:title="updateCardTitle"
/>
```

---

## `v-model` on Components

`v-model` on a component is shorthand for `:modelValue` + `@update:modelValue`.

```html
<!-- Parent -->
<CardTitleInput v-model="card.title" />
<!-- expands to: -->
<CardTitleInput :modelValue="card.title" @update:modelValue="card.title = $event" />
```

```js
// CardTitleInput.vue
export default {
  props: {
    modelValue: { type: String, required: true }
  },
  emits: ['update:modelValue'],

  methods: {
    onInput(e: Event) {
      this.$emit('update:modelValue', (e.target as HTMLInputElement).value)
    }
  }
}
```

```html
<template>
  <input :value="modelValue" @input="onInput" @blur="onInput" />
</template>
```

---

## Multiple `v-model` (Named)

```html
<!-- Parent -->
<CardFilter v-model:priority="filterPriority" v-model:assignee="filterAssignee" />
```

```js
// CardFilter.vue
export default {
  props: {
    priority: { type: String as PropType<Card['priority'] | 'all'>, default: 'all' },
    assignee: { type: String, default: '' }
  },
  emits: ['update:priority', 'update:assignee'],

  methods: {
    setPriority(v: string) { this.$emit('update:priority', v) },
    setAssignee(v: string) { this.$emit('update:assignee', v) }
  }
}
```

---

## `v-model` Modifiers

Built-in: `.lazy` (sync on `change` not `input`), `.number`, `.trim`.

Custom modifiers:

```html
<CardTitleInput v-model.capitalize="title" />
```

```js
export default {
  props: {
    modelValue: String,
    modelModifiers: { default: (): Record<string,boolean> => ({}) }
  },
  emits: ['update:modelValue'],

  methods: {
    onInput(e: Event) {
      let val = (e.target as HTMLInputElement).value
      if (this.modelModifiers.capitalize) {
        val = val.charAt(0).toUpperCase() + val.slice(1)
      }
      this.$emit('update:modelValue', val)
    }
  }
}
```

---

## Props vs provide/inject

| | Props | provide/inject |
|--|-------|---------------|
| Depth | Direct parent → child | Any ancestor → any descendant |
| Visibility | Explicit in template | Implicit — hard to trace |
| Reactivity | Reactive | Manual (computed wrapper) |
| Use when | Normal parent-child data flow | App config, theme, cross-cutting services |
