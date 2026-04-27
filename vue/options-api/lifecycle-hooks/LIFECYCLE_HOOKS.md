# Vue 3 Lifecycle Hooks — Options API

---

## Hook Execution Order

```
beforeCreate()
created()
  │
  ├── beforeMount()
  ├── mounted()          ← DOM ready
  │
  ├── [reactive data changes]
  │     ├── beforeUpdate()
  │     └── updated()
  │
  ├── [component removed]
  │     ├── beforeUnmount()
  │     └── unmounted()   ← cleanup here
  │
  ├── errorCaptured()    ← child errors
  ├── activated()        ← <KeepAlive> enter
  └── deactivated()      ← <KeepAlive> leave
```

---

## 1. `beforeCreate`

Very first hook. Component instance initializing — `data`, `computed`, `methods` not available yet.

```vue
<script>
export default {
  beforeCreate() {
    console.log('Instance creating — no data/methods yet')
    // this.message → undefined
  }
}
</script>
```

**Use for:** plugin setup, rarely needed in app code.

---

## 2. `created`

Instance fully initialized. `data`, `computed`, `methods`, `watch` all ready. DOM not yet created.

```vue
<script>
export default {
  data() {
    return {
      users: [],
      loading: true
    }
  },

  async created() {
    const res = await fetch('/api/users')
    this.users = await res.json()
    this.loading = false
  }
}
</script>
```

**Use for:** API calls, data init, logic that doesn't need DOM. Most common hook for fetching.

---

## 3. `beforeMount`

Render about to happen. DOM not created yet.

```vue
<script>
export default {
  beforeMount() {
    console.log('About to render — DOM not ready')
    // this.$el → not yet available
  }
}
</script>
```

**Use for:** last-chance data prep before first render. Rarely needed.

---

## 4. `mounted`

DOM fully created. `this.$el` and template refs available.

```vue
<script>
export default {
  mounted() {
    this.$refs.input.focus()       // direct DOM access
    this.initChart()               // third-party lib init
  },

  methods: {
    initChart() {
      // e.g. Chart.js, mapbox, etc.
    }
  }
}
</script>

<template>
  <input ref="input" />
</template>
```

**Use for:** DOM access, third-party lib init (charts, maps), event listeners.

---

## 5. `beforeUpdate`

Reactive data changed, DOM re-render about to happen. Old DOM still visible.

```vue
<script>
export default {
  data() {
    return { count: 0 }
  },

  beforeUpdate() {
    // capture scroll position before list re-renders
    this.prevScrollY = window.scrollY
  }
}
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

**Use for:** snapshot DOM state before update (e.g., scroll position).

---

## 6. `updated`

DOM re-rendered after reactive data changed.

```vue
<script>
export default {
  data() {
    return { list: ['a', 'b'] }
  },

  updated() {
    console.log('DOM updated, list length:', this.list.length)
    // sync third-party widget with new DOM
  }
}
</script>
```

> **Warning:** Don't mutate reactive data here — causes infinite loop.

**Use for:** post-render DOM reads, syncing third-party widgets after DOM change.

---

## 7. `beforeUnmount`

Component still fully functional, about to be destroyed.

```vue
<script>
export default {
  data() {
    return { timer: null }
  },

  mounted() {
    this.timer = setInterval(() => console.log('tick'), 1000)
  },

  beforeUnmount() {
    clearInterval(this.timer)   // cleanup while APIs still work
  }
}
</script>
```

**Use for:** final cleanup while `this` and component APIs still accessible.

---

## 8. `unmounted`

Component fully removed from DOM. All refs and child components gone.

```vue
<script>
export default {
  mounted() {
    window.addEventListener('resize', this.handleResize)
  },

  unmounted() {
    window.removeEventListener('resize', this.handleResize)
  },

  methods: {
    handleResize() {
      console.log('width:', window.innerWidth)
    }
  }
}
</script>
```

**Use for:** remove event listeners, cancel subscriptions, disconnect WebSockets.

---

## 9. `errorCaptured`

Catches errors from any descendant component.

```vue
<script>
export default {
  data() {
    return { error: null }
  },

  errorCaptured(err, instance, info) {
    this.error = err.message
    console.error('Caught from child:', err, info)
    return false   // false = stop error propagation
  }
}
</script>

<template>
  <div v-if="error">Error: {{ error }}</div>
  <ChildComponent v-else />
</template>
```

**Use for:** error boundaries around risky child trees.

---

## 10. `activated` / `deactivated`

Only fires for components inside `<KeepAlive>`.

```vue
<!-- Parent -->
<template>
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

```vue
<!-- Child component -->
<script>
export default {
  activated() {
    console.log('Tab shown — resume polling')
    this.startPolling()
  },

  deactivated() {
    console.log('Tab hidden — pause polling')
    this.stopPolling()
  },

  methods: {
    startPolling() { /* ... */ },
    stopPolling() { /* ... */ }
  }
}
</script>
```

**Use for:** pause/resume side effects when switching tabs without destroying component.

---

## 11. `serverPrefetch` (SSR Only)

Called before component renders **on the server**. Resolves async data so HTML is pre-filled on first load.

```vue
<script>
export default {
  data() {
    return { post: null }
  },

  async serverPrefetch() {
    // runs on server only — populates state before HTML is sent to browser
    this.post = await fetchPost(this.$route.params.id)
  },

  async mounted() {
    // fallback — runs on client if not server-rendered
    if (!this.post) {
      this.post = await fetchPost(this.$route.params.id)
    }
  }
}
</script>
```

**Use for:** SSR data fetching (Nuxt, Quasar SSR). No-op in client-only apps.

---

## 13. `renderTracked` / `renderTriggered` (Dev Only)

Debug which reactive dependency triggered a re-render.

```vue
<script>
export default {
  renderTracked(event) {
    console.log('Dependency tracked:', event)
  },

  renderTriggered(event) {
    console.log('Re-render triggered by:', event)
    // event.key, event.target, event.type
  }
}
</script>
```

**Use for:** debugging unexpected re-renders. Stripped in production builds.

---

## Real-World Pattern: Data Fetch + Cleanup

```vue
<script>
export default {
  data() {
    return {
      data: null,
      loading: true,
      controller: null
    }
  },

  async mounted() {
    this.controller = new AbortController()
    try {
      const res = await fetch('/api/data', { signal: this.controller.signal })
      this.data = await res.json()
    } catch (e) {
      if (e.name !== 'AbortError') console.error(e)
    } finally {
      this.loading = false
    }
  },

  unmounted() {
    this.controller?.abort()   // cancel in-flight request if component removed
  }
}
</script>
```

---

## Quick Reference

| Hook | When | Common Use |
|------|------|------------|
| `beforeCreate` | Instance initializing | Plugin setup |
| `created` | Instance ready, no DOM | API calls, data init |
| `beforeMount` | Before first render | Last data prep |
| `mounted` | DOM ready | DOM access, lib init |
| `beforeUpdate` | Before re-render | Snapshot DOM |
| `updated` | After re-render | Post-render DOM reads |
| `beforeUnmount` | Before destroy | Cleanup while APIs live |
| `unmounted` | After destroy | Remove listeners, cancel subs |
| `errorCaptured` | Child error | Error boundaries |
| `activated` | KeepAlive shown | Resume effects |
| `deactivated` | KeepAlive hidden | Pause effects |
| `serverPrefetch` | SSR: before server render | Pre-fill data for SSR |
| `renderTracked` | Dev: dep tracked | Debug reactivity |
| `renderTriggered` | Dev: re-render | Debug re-renders |
