# Lifecycle Hooks — Options API

---

## Execution Order

```
new component instance
  ↓
beforeCreate   — instance created, reactivity not set up
created        — data/computed/watch/methods ready, no DOM
  ↓
beforeMount    — render function about to run
  [child components mount here]
mounted        — this component's DOM is in the document
  ↓
[reactive state changes]
beforeUpdate   — DOM about to re-render
updated        — DOM re-rendered
  ↓
[component removed]
beforeUnmount  — still fully functional
unmounted      — watchers stopped, child components gone
```

**Parent vs child mount order:** parent `beforeMount` fires, then all children mount completely (their `mounted` fires), then parent `mounted` fires. So by the time parent's `mounted` runs, child refs are guaranteed to exist.

---

## `beforeCreate`

Fires before Vue sets up reactivity on the instance. `this.data` properties don't exist yet.

Rarely useful. Main use: plugins that need to intercept before reactivity.

```js
export default {
  beforeCreate() {
    // this.columns → undefined (data not set up)
    // Only useful for low-level plugin initialization
  }
}
```

---

## `created`

Data, computed, methods, and watchers are all ready. DOM doesn't exist yet (`this.$el` is undefined).

**Best place to fetch data** — you're not blocking the browser's layout thread, and you can start fetching before the DOM renders.

```js
export default {
  data() {
    return { columns: [] as Column[], loading: true, error: null as string | null }
  },

  async created() {
    try {
      this.columns = await api.getColumns(this.$route.params.id as string)
    } catch (e) {
      this.error = (e as Error).message
    } finally {
      this.loading = false
    }
  }
}
```

**created vs mounted for data fetch:** `created` fires slightly earlier (before DOM), which doesn't matter for API calls. But if your fetch needs `this.$refs` or DOM dimensions, use `mounted`.

---

## `mounted`

The component's DOM is in the document. `this.$el` is the root element. `this.$refs` are populated.

```js
export default {
  mounted() {
    // DOM access
    const el = this.$refs.columnContainer as HTMLElement
    el.scrollLeft = el.scrollWidth   // scroll to end

    // third-party lib that needs a real DOM node
    this.sortable = new Sortable(el, {
      onEnd: ({ item, to, newIndex }) => this.handleCardDrop(item, to, newIndex)
    })

    // event listeners that need to be cleaned up
    window.addEventListener('keydown', this.handleKeyboard)
  },

  beforeUnmount() {
    window.removeEventListener('keydown', this.handleKeyboard)
    this.sortable?.destroy()
  }
}
```

---

## `beforeMount`

Runs right before the component's render function executes for the first time. DOM not in document yet, but the render is about to happen.

Rarely needed. One edge case: you need to do something after `created` but before the first paint (e.g., SSR hydration prep).

---

## `beforeUpdate`

Fires after reactive state changes, before the DOM re-renders. The DOM still reflects the old state.

```js
export default {
  beforeUpdate() {
    // snapshot the scroll position before the list re-renders
    // so you can restore it after (useful for infinite scroll)
    const list = this.$refs.cardList as HTMLElement
    this._prevScrollTop = list?.scrollTop
  },

  updated() {
    const list = this.$refs.cardList as HTMLElement
    if (list) list.scrollTop = this._prevScrollTop
  }
}
```

---

## `updated`

Fires after the DOM has re-rendered. `this.$el` reflects the new state.

**Critical gotcha:** mutating reactive state here causes an infinite loop.

```js
export default {
  updated() {
    // BAD — triggers re-render → updated → re-render → ...
    this.cardCount = this.columns.flatMap(c => c.cards).length

    // GOOD — use a computed instead
    // computed: { cardCount() { return ... } }

    // OK — DOM read that doesn't affect reactive state
    const height = this.$el.offsetHeight
    this.reportHeight?.(height)  // only if reportHeight is an external callback
  }
}
```

**When to actually use `updated`:** syncing a third-party widget (like a chart) that needs to read the updated DOM. Even then, prefer `watch` on the data that drives the DOM change.

---

## `beforeUnmount`

Component still fully functional — `this` works, refs work, watchers run. Last chance to clean up before teardown.

```js
export default {
  mounted() {
    this._resizeObserver = new ResizeObserver(this.handleResize)
    this._resizeObserver.observe(this.$el)
    document.addEventListener('visibilitychange', this.handleVisibility)
  },

  beforeUnmount() {
    this._resizeObserver?.disconnect()
    document.removeEventListener('visibilitychange', this.handleVisibility)
  }
}
```

---

## `unmounted`

Everything is torn down — watchers stopped, child components destroyed, DOM removed. `this.$el` is gone.

Mostly used when the cleanup itself is async or when you're working with resources outside Vue's control.

```js
export default {
  mounted() {
    this.ws = new WebSocket('wss://api.example.com/board')
    this.ws.onmessage = ({ data }) => {
      const update = JSON.parse(data)
      this.applyBoardUpdate(update)
    }
  },

  unmounted() {
    this.ws?.close()
  }
}
```

**beforeUnmount vs unmounted:** prefer `beforeUnmount` — Vue still has everything set up. `unmounted` is for when you need to wait until Vue has finished tearing down (rare).

---

## `activated` / `deactivated`

Only fires for components inside `<KeepAlive>`. The component stays alive in memory rather than being destroyed — `mounted`/`unmounted` don't fire again on tab switch.

```js
export default {
  data() { return { pollingTimer: null as ReturnType<typeof setInterval> | null } },

  activated() {
    // user switched back to this board tab
    this.refreshBoard()
    this.pollingTimer = setInterval(this.refreshBoard, 30_000)
  },

  deactivated() {
    // user switched away — don't waste requests
    clearInterval(this.pollingTimer!)
    this.pollingTimer = null
  }
}
```

**Why not mounted/unmounted?** With KeepAlive, those only fire once (on first mount / final unmount). `activated`/`deactivated` fire every time the component enters/leaves the cache.

---

## `errorCaptured`

Catches errors thrown in any descendant component's setup, lifecycle hooks, or event handlers. Return `false` to stop propagation to parent error handlers.

```js
export default {
  data() { return { boardError: null as string | null } },

  errorCaptured(err: Error, instance: ComponentPublicInstance | null, info: string) {
    this.boardError = `Failed to load: ${err.message}`
    console.error('[BoardView] child error:', err, info)
    return false   // don't bubble to app-level error handler
  }
}
```

---

## `serverPrefetch`

SSR only (Nuxt, etc.). Called on the server before rendering. Use it to pre-populate data so the HTML sent to the browser is fully hydrated.

```js
export default {
  async serverPrefetch() {
    // runs on server — populates store before HTML is serialized
    await useBoardStore().loadBoard(this.$route.params.id)
  },

  async mounted() {
    // runs on client — data already hydrated from server, skip if present
    if (!useBoardStore().columns.length) {
      await useBoardStore().loadBoard(this.$route.params.id)
    }
  }
}
```

---

## `renderTracked` / `renderTriggered` (Dev Only)

Debug which reactive dependency caused a re-render. Stripped in production.

```js
export default {
  renderTriggered({ key, target, type }) {
    console.log(`Re-render triggered: ${String(key)} was ${type}`)
    // "Re-render triggered: cards was set"
  }
}
```

---

## Hook Execution Order (Parent + Child)

```
parent beforeCreate
parent created
parent beforeMount
  child beforeCreate
  child created
  child beforeMount
  child mounted          ← child fully mounted first
parent mounted           ← then parent

[state changes]
parent beforeUpdate
  child beforeUpdate
  child updated
parent updated

[teardown]
parent beforeUnmount
  child beforeUnmount
  child unmounted
parent unmounted
```

---

## Quick Reference

| Hook | DOM | `this.$refs` | Common use |
|------|-----|-------------|------------|
| `beforeCreate` | ✗ | ✗ | Plugin setup |
| `created` | ✗ | ✗ | API fetch, data init |
| `beforeMount` | ✗ | ✗ | Rare |
| `mounted` | ✓ | ✓ | DOM libs, event listeners |
| `beforeUpdate` | old DOM | old | Scroll snapshot |
| `updated` | ✓ | ✓ | Sync 3rd-party widget |
| `beforeUnmount` | ✓ | ✓ | Cleanup (preferred) |
| `unmounted` | ✗ | ✗ | Async cleanup |
| `activated` | ✓ | ✓ | Resume polling (KeepAlive) |
| `deactivated` | ✓ | ✓ | Pause polling (KeepAlive) |
| `errorCaptured` | — | — | Error boundary |
| `serverPrefetch` | — | — | SSR hydration |
