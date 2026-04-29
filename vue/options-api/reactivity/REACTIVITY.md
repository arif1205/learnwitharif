# Reactivity — Options API

---

## How Vue Reactivity Works

Vue wraps `data()` return value in a `Proxy`. Any property get registers a dependency; any property set triggers re-render of components that depend on it. This is why you can't add new reactive properties after initialization — the Proxy only intercepts properties that existed at creation.

```
data() return value  →  Vue Proxy  →  dep tracking  →  re-render on set
```

---

## `data()`

Declare all reactive state here. Properties added after initialization (`this.newProp = x`) are **not reactive**.

```js
export default {
  data() {
    return {
      columns: [] as Column[],
      activeColumnId: null as number | null,
      dragState: {
        cardId: null as number | null,
        sourceColumnId: null as number | null
      },
      loading: false,
      error: null as string | null
    }
  }
}
```

**Gotcha — nested object mutation:** Vue tracks the reference, not deeply by default for performance. Mutating nested objects works because the Proxy intercepts property sets at every level (Vue 3 uses recursive proxy). But replacing the whole nested object with a new reference also works.

```js
// both work in Vue 3
this.dragState.cardId = 5           // triggers reactivity
this.dragState = { cardId: 5, sourceColumnId: 1 }  // also works
```

**Gotcha — arrays:** Vue 3 intercepts array methods (`push`, `splice`, etc.) via Proxy, so they're reactive. Index assignment also works.

```js
this.columns.push(newColumn)    // reactive ✓
this.columns[0] = newColumn     // reactive ✓ (Vue 3 only — Vue 2 needed $set)
```

---

## `computed`

Cached derived values. Re-evaluates only when its reactive dependencies change. Accessing the same computed multiple times in a template only runs the function once.

```js
export default {
  data() {
    return { columns: [] as Column[], filterPriority: 'high' as Card['priority'] | 'all' }
  },

  computed: {
    totalCards(): number {
      return this.columns.reduce((n, col) => n + col.cards.length, 0)
    },

    highPriorityCards(): Card[] {
      return this.columns
        .flatMap(col => col.cards)
        .filter(card => card.priority === 'high')
    },

    filteredColumns(): Column[] {
      if (this.filterPriority === 'all') return this.columns
      return this.columns.map(col => ({
        ...col,
        cards: col.cards.filter(c => c.priority === this.filterPriority)
      }))
    },

    // writable computed — e.g. sync two-way with parent
    boardTitle: {
      get(): string  { return this.board?.title ?? '' },
      set(v: string) { this.$emit('update:title', v) }
    }
  }
}
```

**computed vs method:** A method called in a template recalculates on every re-render. A computed recalculates only when its deps change. For anything non-trivial (array filter, format operations), always use computed.

---

## `watch`

Run side effects when reactive state changes. More explicit than `watchEffect` — you declare what you're watching.

```js
export default {
  data() {
    return {
      searchQuery: '',
      boardId: null as number | null,
      columns: [] as Column[]
    }
  },

  watch: {
    // simple handler
    searchQuery(newVal: string, oldVal: string) {
      if (newVal !== oldVal) this.filterCards(newVal)
    },

    // immediate — runs on component creation too, good for initial data fetch
    boardId: {
      async handler(id: number | null) {
        if (!id) return
        this.loading = true
        this.columns = await api.getColumns(id)
        this.loading = false
      },
      immediate: true
    },

    // deep — watches nested changes in objects/arrays
    columns: {
      handler(cols: Column[]) {
        localStorage.setItem(`board-${this.boardId}`, JSON.stringify(cols))
      },
      deep: true
    },

    // dot-notation — watch specific nested property (cheaper than deep)
    'dragState.cardId'(cardId: number | null) {
      document.body.classList.toggle('is-dragging', cardId !== null)
    }
  }
}
```

**watch vs computed:** computed = derive a value. watch = run a side effect (API call, localStorage, DOM manipulation, emit to parent).

**deep watch cost:** Deep watch traverses the entire object tree to register deps. On large arrays (100+ items), this has overhead. Watch specific properties when possible.

---

## `methods`

Plain functions on the component instance. Not cached. Called on every invocation. Use for event handlers, imperative logic, and anything that shouldn't be cached.

```js
export default {
  methods: {
    addCard(columnId: number, title: string): void {
      const col = this.columns.find(c => c.id === columnId)
      if (!col) return
      col.cards.push({ id: Date.now(), title, priority: 'medium', columnId })
    },

    moveCard(cardId: number, fromColumnId: number, toColumnId: number): void {
      const from = this.columns.find(c => c.id === fromColumnId)
      const to   = this.columns.find(c => c.id === toColumnId)
      if (!from || !to) return
      const idx  = from.cards.findIndex(c => c.id === cardId)
      const [card] = from.cards.splice(idx, 1)
      card.columnId = toColumnId
      to.cards.push(card)
    },

    async saveBoard(): Promise<void> {
      try {
        await api.saveColumns(this.boardId, this.columns)
      } catch (e) {
        this.error = (e as Error).message
      }
    }
  }
}
```

**Arrow functions:** Don't use arrow functions for methods — they capture `this` from the enclosing scope (the module), not the component instance.

```js
// BAD
methods: {
  addCard: (columnId) => {  // `this` is undefined or window
    this.columns  // ✗
  }
}
```

---

## `$watch` — Programmatic Watcher

When you need to conditionally start/stop watching, or watch something computed at runtime.

```js
export default {
  mounted() {
    // watch a computed expression
    this._unwatchCards = this.$watch(
      () => this.columns.reduce((n, c) => n + c.cards.length, 0),
      (newTotal) => {
        if (newTotal > 100) this.showWarning = true
      }
    )
  },

  beforeUnmount() {
    this._unwatchCards?.()   // stop watcher to prevent memory leak
  }
}
```

---

## Reactivity Utilities

```js
import { markRaw, toRaw } from 'vue'

export default {
  data() {
    return {
      // markRaw — third-party objects that Vue shouldn't proxy
      // avoids Proxy overhead + prevents lib's internal reference checks breaking
      sortableInstance: markRaw(null as Sortable | null),
      chartInstance: markRaw(null as Chart | null)
    }
  },

  methods: {
    initSortable(el: HTMLElement) {
      this.sortableInstance = markRaw(new Sortable(el, { onEnd: this.handleDrop }))
    },

    exportBoard() {
      // toRaw strips the Proxy for clean JSON serialization
      return JSON.stringify(toRaw(this.columns))
    }
  }
}
```

---

## `$nextTick`

Vue batches DOM updates — after you mutate state, the DOM hasn't updated yet. `$nextTick` gives you a callback/promise after the next DOM flush.

```js
export default {
  methods: {
    async addCardAndFocus(columnId: number) {
      const col = this.columns.find(c => c.id === columnId)!
      col.cards.push({ id: Date.now(), title: '', priority: 'medium', columnId })

      await this.$nextTick()   // DOM now has the new card element
      const inputs = this.$el.querySelectorAll('.card-input')
      inputs[inputs.length - 1]?.focus()
    }
  }
}
```

---

## Common Mistakes

**1. Watching without `deep: true` on objects:**
```js
watch: {
  columns(v) { /* won't fire when cards inside change — only on array replace */ }
  // fix: deep: true, or watch 'columns[n].cards'
}
```

**2. Mutating state in `updated`:**
```js
updated() {
  this.count++  // triggers another update → infinite loop
  // fix: use watch with a condition, or nextTick
}
```

**3. Expecting new properties to be reactive:**
```js
mounted() {
  this.newFlag = true  // NOT reactive — not in data()
  // fix: declare in data(), or use Vue.set (Vue 2), or restructure
}
```

**4. Performance — deep watching large arrays:**
```js
// BAD — deep watches all 500 cards
watch: { columns: { handler: saveToStorage, deep: true } }

// BETTER — watch specific computed result
watch: {
  serializedBoard(v) { localStorage.setItem('board', v) }
}
computed: {
  serializedBoard() { return JSON.stringify(this.columns) }
}
```
