# Vue 3 Complete Cheat Sheet — Options API

---

## Project Boot Sequence

```
index.html → <div id="app"> → main.ts → createApp(App).mount('#app')
```

## main.ts Full Setup

```ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import router from './router'
import App from './App.vue'

const app = createApp(App)

app.use(createPinia())
app.use(router)
app.component('BaseButton', BaseButton)        // global component
app.provide('apiUrl', 'https://api.com')       // DI
app.directive('focus', { mounted: el => el.focus() })
app.config.errorHandler = (err) => logError(err)
app.config.globalProperties.$filters = { currency: v => `$${v.toFixed(2)}` }

app.mount('#app')   // always last
```

---

## Single File Component (SFC)

```vue
<script>
export default {
  name: 'MyComponent',

  data() {
    return {
      // reactive state
    }
  },

  computed: {},
  watch: {},
  methods: {},

  // lifecycle hooks
  mounted() {}
}
</script>

<template>
  <!-- HTML + Vue syntax -->
</template>

<style scoped>
/* CSS scoped to this component */
</style>
```

---

## Reactivity

```js
export default {
  data() {
    return {
      count: 0,
      board: { title: '', columns: [] }
    }
  },

  computed: {
    // cached — recalculates when deps change
    total() {
      return this.board.columns.length
    },
    // writable computed
    fullName: {
      get() { return `${this.first} ${this.last}` },
      set(val) { [this.first, this.last] = val.split(' ') }
    }
  },

  watch: {
    // fires when count changes
    count(newVal, oldVal) {
      console.log(newVal)
    },
    // deep + immediate
    board: {
      handler(newVal) { console.log(newVal) },
      deep: true,
      immediate: true
    }
  },

  methods: {
    increment() {
      this.count++
    }
  }
}
```

---

## Template Syntax

```html
{{ variable }}                        <!-- display value -->
:attr="value"                         <!-- bind JS to attribute -->
@click="handler"                      <!-- event listener -->
@click.prevent="handler"              <!-- event modifier -->
@keyup.enter="submit"                 <!-- key modifier -->
v-if="condition"                      <!-- conditional render -->
v-else-if="other"
v-else
v-show="condition"                    <!-- toggle display:none -->
v-for="item in items" :key="item.id"  <!-- loop -->
v-model="value"                       <!-- two-way binding -->
v-once                                <!-- render once, skip updates -->
```

---

## Props + Emits

```vue
<!-- Child -->
<script>
export default {
  props: {
    title: { type: String, required: true },
    count: { type: Number, default: 0 }
  },

  emits: ['addCard', 'delete'],

  methods: {
    handleAdd() {
      this.$emit('addCard', 1, 'New Task')
    }
  }
}
</script>
```

```vue
<!-- Parent -->
<Child :title="name" :count="5" @add-card="handler" />
```

---

## v-model on Custom Components

```vue
<!-- Parent -->
<CardInput v-model="title" />
<!-- expands to: :modelValue="title" @update:modelValue="title = $event" -->

<!-- Child: CardInput.vue -->
<script>
export default {
  props: {
    modelValue: { type: String }
  },
  emits: ['update:modelValue']
}
</script>
<template>
  <input :value="modelValue" @input="$emit('update:modelValue', $event.target.value)" />
</template>
```

---

## Lifecycle Hooks

```js
export default {
  beforeCreate() {},    // instance initializing
  created() {},         // data/methods ready — fetch here
  beforeMount() {},     // before DOM created
  mounted() {},         // DOM ready — access refs, init libs
  beforeUpdate() {},    // before re-render
  updated() {},         // after re-render
  beforeUnmount() {},   // cleanup while APIs still live
  unmounted() {},       // cleanup — remove listeners, cancel subs
  activated() {},       // KeepAlive — came into view
  deactivated() {}      // KeepAlive — hidden
}
```

See [lifecycle-hooks/LIFECYCLE_HOOKS.md](lifecycle-hooks/LIFECYCLE_HOOKS.md) for full examples.

---

## Template Refs

```vue
<script>
export default {
  mounted() {
    this.$refs.input.focus()   // access after mounted
  }
}
</script>

<template>
  <input ref="input" />
</template>
```

Child component ref — call exposed methods directly:

```vue
<!-- Parent -->
<script>
export default {
  mounted() {
    this.$refs.card.focus()   // call child method
  }
}
</script>

<template>
  <CardComponent ref="card" />
</template>
```

---

## Slots

```vue
<!-- Basic -->
<BaseCard><p>Content</p></BaseCard>
<template><div><slot /></div></template>

<!-- Named -->
<Layout>
  <template #header><h1>Title</h1></template>
  <template #default>Main</template>
  <template #footer>Footer</template>
</Layout>
<template>
  <slot name="header" />
  <slot />
  <slot name="footer" />
</template>

<!-- Scoped — child passes data to parent's slot -->
<slot :card="card" :index="i" />
<Child><template #default="{ card, index }">{{ card.title }}</template></Child>
```

---

## Mixins (Reuse Logic)

Mixins are Options API's code-reuse mechanism. Prefer composables via `setup()` for new code.

```js
// src/mixins/editableMixin.js
export const editableMixin = {
  data() {
    return { isEditing: false, editText: '' }
  },
  methods: {
    startEdit(title) {
      this.editText = title
      this.isEditing = true
    },
    cancelEdit() {
      this.isEditing = false
    }
  }
}

// in component
import { editableMixin } from '@/mixins/editableMixin'

export default {
  mixins: [editableMixin],
  // this.isEditing, this.startEdit() available
}
```

---

## provide / inject (Dependency Injection)

```js
// Provider component
export default {
  provide() {
    return {
      theme: this.theme   // not reactive by default
    }
  },
  data() {
    return { theme: 'dark' }
  }
}

// For reactive provide, use computed
import { computed } from 'vue'
export default {
  provide() {
    return {
      theme: computed(() => this.theme)
    }
  }
}
```

```js
// Consumer — any descendant, any depth
export default {
  inject: ['theme'],
  // this.theme available

  // with default value
  inject: {
    theme: { default: 'light' }
  }
}
```

---

## Directives

```ts
// Global
app.directive('focus', { mounted: (el) => el.focus() })
app.directive('click-outside', {
  mounted(el, binding) {
    el._handler = (e) => { if (!el.contains(e.target)) binding.value() }
    document.addEventListener('click', el._handler)
  },
  unmounted(el) { document.removeEventListener('click', el._handler) }
})

// Usage
// <input v-focus />
// <div v-click-outside="closeMenu">...</div>
```

---

## Teleport — render outside component tree

```vue
<Teleport to="body">
  <Modal v-if="showModal" />   <!-- renders directly under <body> -->
</Teleport>
```

---

## KeepAlive — cache component state

```vue
<KeepAlive :max="5">
  <RouterView />
</KeepAlive>
```

---

## Dynamic Component

```js
export default {
  data() {
    return {
      active: 'board',
      views: { board: BoardView, settings: SettingsView }
    }
  }
}
```

```html
<component :is="views[active]" />
```

---

## Async Components

```ts
import { defineAsyncComponent } from 'vue'

const HeavyChart = defineAsyncComponent({
  loader: () => import('./HeavyChart.vue'),
  loadingComponent: Spinner,
  errorComponent: ErrorMsg,
  delay: 200,
  timeout: 5000,
})
```

---

## nextTick — wait for DOM update

```js
export default {
  methods: {
    async addAndFocus() {
      this.cards.push(newCard)
      await this.$nextTick()      // DOM updated
      this.$refs.newInput.focus() // safe to access new element
    }
  }
}
```

---

## Component Options

```js
export default {
  name: 'KanbanCard',             // for DevTools + recursive components
  inheritAttrs: false,            // don't auto-apply parent attrs to root
  components: { ChildComponent }, // local registration
}
```

```html
<!-- forward attrs manually to specific element -->
<input v-bind="$attrs" />
```

---

## Transitions

```vue
<Transition name="fade">
  <div v-if="show">...</div>
</Transition>

<TransitionGroup name="card" tag="div">
  <Card v-for="c in cards" :key="c.id" />
</TransitionGroup>
```

```css
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
.card-move { transition: transform 0.3s; }
```

---

## Vue Router

```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: AppLayout,
      children: [
        { path: '', component: HomeView },
        { path: 'board/:id', name: 'board', component: () => import('../views/BoardView.vue') },
      ]
    },
    { path: '/login', component: LoginView, meta: { requiresAuth: false } },
    { path: '/:pathMatch(.*)*', component: NotFoundView },
  ],
  scrollBehavior(to, from, saved) {
    return saved ?? { top: 0 }
  }
})

// global guard
router.beforeEach((to) => {
  const auth = useAuthStore()
  if (to.meta.requiresAuth && !auth.isLoggedIn) return { path: '/login' }
})
```

```js
// in component — Options API
export default {
  computed: {
    boardId() { return this.$route.params.id }
  },

  methods: {
    goToBoard(id) {
      this.$router.push({ name: 'board', params: { id } })
    },
    goBack() {
      this.$router.back()
    }
  },

  // in-component route guards
  beforeRouteEnter(to, from, next) {
    next(vm => { /* vm = component instance */ })
  },
  beforeRouteUpdate(to, from) {
    // fires when same component but route params change
    this.fetchData(to.params.id)
  },
  beforeRouteLeave(to, from) {
    if (this.unsaved) return false   // block navigation
  }
}
```

---

## Pinia

```ts
// stores/board.ts — setup store (works with Options API too)
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useBoardStore = defineStore('board', () => {
  const columns = ref([])

  const totalCards = computed(() =>
    columns.value.reduce((n, col) => n + col.cards.length, 0)
  )

  function addCard(columnId, title) {
    const col = columns.value.find(c => c.id === columnId)
    col?.cards.push({ id: Date.now(), title })
  }

  return { columns, totalCards, addCard }
})
```

```js
// in Options API component
import { mapStores, mapState, mapActions } from 'pinia'
import { useBoardStore } from '@/stores/board'

export default {
  computed: {
    ...mapStores(useBoardStore),                         // this.boardStore
    ...mapState(useBoardStore, ['columns', 'totalCards']) // this.columns, this.totalCards
  },

  methods: {
    ...mapActions(useBoardStore, ['addCard'])             // this.addCard()
  }
}
```

---

## App Instance API

```ts
app.use(plugin)
app.component('Name', Component)
app.directive('name', hooks)
app.provide(key, value)
app.config.globalProperties.$x = value
app.config.errorHandler = (err, instance, info) => {}
app.config.warnHandler = (msg) => {}
app.config.compilerOptions.isCustomElement = tag => tag.startsWith('my-')
app.config.performance = true
app.unmount()
app.version   // "3.5.x"
```
