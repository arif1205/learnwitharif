# Vue 3 — Options API Master Reference

All topics. Each section: what it does, one code block, key "vs" note. Links to deep-dives.

---

## Domain Types (used throughout)

```ts
interface Card   { id: number; title: string; priority: 'low'|'medium'|'high'; columnId: number }
interface Column { id: number; title: string; cards: Card[]; boardId: number }
interface Board  { id: number; title: string; columns: Column[] }
```

---

## SFC Structure

Three blocks: `<script>` for logic, `<template>` for DOM, `<style scoped>` for CSS. Vue compiles SFCs at build time — scoped styles add a unique `data-v-xxxx` attribute.

```vue
<script>
export default {
  name: 'KanbanCard',
  data()    { return { isEditing: false } },
  computed: { /* derived */ },
  watch:    { /* side effects */ },
  methods:  { /* handlers */ },
  mounted() { /* DOM ready */ }
}
</script>
<template><div /></template>
<style scoped>.card { /* isolated */ }</style>
```

---

## Reactivity

`data()` returns reactive state. Vue wraps it in a Proxy — any set triggers re-render. `computed` is cached until deps change. `watch` runs side effects imperatively.

```js
export default {
  data() {
    return { columns: [] as Column[], loading: false }
  },
  computed: {
    totalCards(): number { return this.columns.reduce((n, c) => n + c.cards.length, 0) },
  },
  watch: {
    'columns': { handler(v) { localStorage.setItem('board', JSON.stringify(v)) }, deep: true }
  }
}
```

**computed vs method:** computed caches — same result until deps change. Method recalculates every render call.
**watch vs computed:** use `computed` when you need a value. Use `watch` when you need a side effect (API call, localStorage, DOM).

**→ [reactivity/REACTIVITY.md](reactivity/REACTIVITY.md)**

---

## Template Syntax

Directives are compiler macros — `:` = `v-bind`, `@` = `v-on`. `v-if` removes the node; `v-show` sets `display:none`. `:key` is required on `v-for` — Vue uses it for VDOM diffing.

```html
<div v-for="col in columns" :key="col.id" :class="{ 'col--dragging': dragging === col.id }">
  <h2>{{ col.title }} ({{ col.cards.length }})</h2>
  <button @click.stop="addCard(col.id)">+</button>
  <KanbanCard v-for="card in col.cards" :key="card.id" :card="card" />
</div>
```

**v-if vs v-show:** `v-if` for infrequent toggles (expensive mount/unmount). `v-show` for frequent toggles (modal, tooltip).

**→ [template-syntax/TEMPLATE_SYNTAX.md](template-syntax/TEMPLATE_SYNTAX.md)**

---

## Lifecycle Hooks

`created` = data ready, no DOM. `mounted` = DOM ready, refs available. `unmounted` = cleanup. Each fires on the current component instance — parent `mounted` fires after all child `mounted`.

```js
export default {
  async created()     { this.columns = await fetchColumns(this.boardId) },
  mounted()           { this.$refs.firstColumn?.scrollIntoView() },
  beforeUnmount()     { window.removeEventListener('keydown', this.handleKey) },
  activated()         { this.startPolling() },   // KeepAlive
  deactivated()       { this.stopPolling() },
}
```

**→ [lifecycle-hooks/LIFECYCLE_HOOKS.md](lifecycle-hooks/LIFECYCLE_HOOKS.md)**

---

## Props & Emits

Props flow down, events flow up. Declare both explicitly — Vue uses `emits` to distinguish component events from native DOM events and for tree-shaking.

```js
export default {
  props: {
    card:     { type: Object as PropType<Card>, required: true },
    readonly: { type: Boolean, default: false }
  },
  emits: {
    'update:title': (title: string) => title.length > 0,
    'move':         (columnId: number) => true
  },
  methods: {
    saveTitle(t: string) { this.$emit('update:title', t) }
  }
}
```

**→ [props-emits/PROPS_EMITS.md](props-emits/PROPS_EMITS.md)**

---

## Slots

Slots let parent inject DOM into child's template. Scoped slots are the opposite — child exposes data to parent's slot template. Think render props from React.

```html
<!-- KanbanColumn.vue -->
<div class="column"><slot name="header" /><slot :cards="cards" /></div>

<!-- usage -->
<KanbanColumn>
  <template #header><h2>In Progress</h2></template>
  <template #default="{ cards }">
    <KanbanCard v-for="c in cards" :key="c.id" :card="c" />
  </template>
</KanbanColumn>
```

**→ [slots/SLOTS.md](slots/SLOTS.md)**

---

## Composables (Reuse Logic)

Composables are functions that encapsulate reactive logic. Options API equivalent is mixins, but composables are explicit about what they return — no naming collisions, easy to test.

```ts
// useCardDrag.ts
export function useCardDrag() {
  const dragging = ref<number | null>(null)
  function startDrag(cardId: number) { dragging.value = cardId }
  function endDrag()                  { dragging.value = null }
  return { dragging, startDrag, endDrag }
}

// in component via setup()
export default {
  setup() {
    return { ...useCardDrag() }
  }
}
```

**composables vs mixins:** mixins merge silently — conflicts are invisible. Composables return explicitly. Prefer composables.

**→ [composables/COMPOSABLES.md](composables/COMPOSABLES.md)**

---

## provide / inject

Solves prop-drilling for deep trees. `provide` in ancestor, `inject` in any descendant. Not reactive by default — wrap in `computed` if consumer needs live updates.

```js
// BoardView.vue
export default {
  provide() {
    return { boardId: computed(() => this.boardId) }
  }
}

// KanbanCard.vue (5 levels deep)
export default {
  inject: ['boardId']
}
```

**→ [provide-inject/PROVIDE_INJECT.md](provide-inject/PROVIDE_INJECT.md)**

---

## Custom Directives

For imperative DOM manipulation that doesn't belong in component logic. The directive has access to the raw element — use it for focus management, click-outside detection, drag handles.

```ts
app.directive('autofocus', { mounted: (el: HTMLElement) => el.focus() })
// <input v-autofocus />
```

**→ [directives/DIRECTIVES.md](directives/DIRECTIVES.md)**

---

## Transitions

`<Transition>` wraps a single `v-if`/`v-show` element and applies CSS classes during enter/leave. `<TransitionGroup>` does the same for `v-for` lists, plus animates position changes with `[name]-move`.

```html
<TransitionGroup name="card" tag="div" class="column__cards">
  <KanbanCard v-for="card in cards" :key="card.id" :card="card" />
</TransitionGroup>
```
```css
.card-enter-from, .card-leave-to { opacity: 0; transform: translateY(-8px); }
.card-enter-active, .card-leave-active { transition: all .2s ease; }
.card-move { transition: transform .2s ease; }
```

**→ [transitions/TRANSITIONS.md](transitions/TRANSITIONS.md)**

---

## Vue Router

`useRoute` = current route (params, query, meta). `useRouter` = navigation. Guards run in order: global beforeEach → per-route beforeEnter → component beforeRouteEnter.

```js
export default {
  computed: {
    boardId() { return Number(this.$route.params.id) }
  },
  methods: {
    openCard(id: number) {
      this.$router.push({ name: 'card', params: { boardId: this.boardId, cardId: id } })
    }
  },
  beforeRouteLeave() {
    return this.hasUnsavedChanges ? confirm('Discard changes?') : true
  }
}
```

**→ [router/ROUTER.md](router/ROUTER.md)**

---

## Pinia

Global state. `defineStore` with setup-style syntax. In Options API components, use `mapStores`/`mapState`/`mapActions` or `setup()` to expose the store.

```ts
export const useBoardStore = defineStore('board', () => {
  const columns = ref<Column[]>([])
  const totalCards = computed(() => columns.value.reduce((n, c) => n + c.cards.length, 0))
  async function loadBoard(id: number) { columns.value = await api.getColumns(id) }
  return { columns, totalCards, loadBoard }
})
```
```js
// in component
export default {
  computed: { ...mapState(useBoardStore, ['columns', 'totalCards']) },
  methods:  { ...mapActions(useBoardStore, ['loadBoard']) }
}
```

**→ [pinia/PINIA.md](pinia/PINIA.md)**

---

## Advanced

`<Teleport>` renders content outside the component tree (modals, toasts). `<KeepAlive>` caches component state. `<Suspense>` coordinates async loading. Render functions skip the template compiler.

```html
<Teleport to="body">
  <CardDetailModal v-if="activeCard" :card="activeCard" @close="activeCard = null" />
</Teleport>
```

**→ [advanced/ADVANCED.md](advanced/ADVANCED.md)**

---

## Quick Comparisons

| | Use when |
|--|---------|
| `v-if` | Condition rarely changes, or element is expensive |
| `v-show` | Frequent toggle, element always needed |
| `computed` | Derived value from reactive state |
| `watch` | Side effect in response to state change |
| `watch (deep)` | Object/array change, need old vs new |
| `provide/inject` | Cross-tree sharing without prop drilling |
| `Teleport` | Need to escape CSS stacking/overflow |
| `KeepAlive` | Expensive component that needs state preservation |
| composable | Reusable reactive logic (replaces mixin) |
| directive | Imperative raw DOM access |
