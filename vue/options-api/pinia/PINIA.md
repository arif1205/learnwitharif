# Pinia — Options API

---

## Setup

```ts
// main.ts
import { createPinia } from 'pinia'
app.use(createPinia())
```

---

## Define a Store

```ts
// stores/counter.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCounterStore = defineStore('counter', () => {
  // state
  const count = ref(0)
  const name = ref('Arif')

  // getter (computed)
  const doubled = computed(() => count.value * 2)

  // action
  function increment() {
    count.value++
  }

  async function fetchUser(id) {
    const res = await fetch(`/api/users/${id}`)
    name.value = (await res.json()).name
  }

  return { count, name, doubled, increment, fetchUser }
})
```

---

## Use Store in Options API Component

```js
import { mapStores, mapState, mapActions } from 'pinia'
import { useCounterStore } from '@/stores/counter'

export default {
  computed: {
    // access full store as this.counterStore
    ...mapStores(useCounterStore),

    // map individual state/getters
    ...mapState(useCounterStore, ['count', 'name', 'doubled']),

    // map with rename
    ...mapState(useCounterStore, {
      myCount: 'count',
      myName: store => store.name.toUpperCase()
    })
  },

  methods: {
    // map actions
    ...mapActions(useCounterStore, ['increment', 'fetchUser']),

    doSomething() {
      // or access via store instance
      this.counterStore.increment()
      this.counterStore.count = 10   // direct mutation OK in Pinia
    }
  },

  async mounted() {
    await this.fetchUser(1)
  }
}
```

---

## Direct Store Access

```js
import { useCounterStore } from '@/stores/counter'

export default {
  setup() {
    return { store: useCounterStore() }
  },

  computed: {
    count() { return this.store.count }
  },

  methods: {
    increment() { this.store.increment() }
  }
}
```

---

## `$patch` — Batch Updates

```js
// object form
this.counterStore.$patch({ count: 10, name: 'New' })

// function form — for complex mutations
this.counterStore.$patch((state) => {
  state.count++
  state.name = 'Updated'
})
```

---

## `$reset` — Reset to Initial State

```ts
// store must define $reset manually in setup stores
export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)

  function $reset() {
    count.value = 0
  }

  return { count, $reset }
})
```

```js
this.counterStore.$reset()
```

---

## Subscribe to Changes

```js
export default {
  mounted() {
    // watch state changes
    this.counterStore.$subscribe((mutation, state) => {
      console.log('Store changed:', mutation.type, state)
      localStorage.setItem('counter', JSON.stringify(state))
    })

    // watch action calls
    this.counterStore.$onAction(({ name, after, onError }) => {
      after(() => console.log(`${name} completed`))
      onError((err) => console.error(`${name} failed:`, err))
    })
  }
}
```

---

## Persist State (pinia-plugin-persistedstate)

```ts
// main.ts
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'
pinia.use(piniaPluginPersistedstate)
```

```ts
// store
export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  return { count }
}, {
  persist: true               // persist everything
  // persist: { pick: ['count'] }   // persist specific keys
})
```

---

## Quick Reference

| Task | Code |
|------|------|
| Map store instance | `...mapStores(useXStore)` → `this.xStore` |
| Map state/getters | `...mapState(useXStore, ['key'])` → `this.key` |
| Map actions | `...mapActions(useXStore, ['fn'])` → `this.fn()` |
| Batch update | `this.xStore.$patch({ key: val })` |
| Subscribe to changes | `this.xStore.$subscribe(cb)` |
| Watch actions | `this.xStore.$onAction(cb)` |
| Dispose store | `this.xStore.$dispose()` |
