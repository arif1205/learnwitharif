# Vue 3 Complete Cheat Sheet

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
<script setup lang="ts">
// logic
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

```ts
import { ref, reactive, computed, watch, watchEffect } from 'vue'

// primitives
const count = ref(0)
count.value++                        // .value in JS
// {{ count }} in template — no .value needed

// objects
const board = reactive({ title: '', columns: [] })
board.title = 'My Board'             // no .value

// computed — cached, recalculates when deps change
const total = computed(() => columns.value.length)

// watch — explicit, fires on change
watch(count, (newVal, oldVal) => { console.log(newVal) })
watch([count, title], ([newCount, newTitle]) => {})
watch(count, cb, { deep: true, immediate: true })

// watchEffect — auto-tracks, runs immediately
watchEffect(() => {
  document.title = `${count.value} items`
})
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
<script setup lang="ts">
const props = defineProps<{
  title: string
  count?: number
}>()

withDefaults(defineProps<{ count?: number }>(), { count: 0 })

const emit = defineEmits<{
  addCard: [columnId: number, title: string]
  delete: [id: number]
}>()

emit('addCard', 1, 'New Task')
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
<script setup>
defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
</script>
<template>
  <input :value="modelValue" @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)" />
</template>
```

---

## Lifecycle Hooks

```ts
import { onBeforeMount, onMounted, onUpdated, onUnmounted, onActivated, onDeactivated } from 'vue'

onBeforeMount(() => {})   // before DOM created
onMounted(() => {})       // DOM ready — fetch data, init libs
onUpdated(() => {})       // after re-render
onUnmounted(() => {})     // cleanup — clear intervals, remove listeners
onActivated(() => {})     // KeepAlive — came into view
onDeactivated(() => {})   // KeepAlive — hidden
```

---

## Template Refs

```vue
<script setup>
const inputEl = ref<HTMLInputElement>()
onMounted(() => inputEl.value?.focus())
</script>

<template>
  <input ref="inputEl" />
</template>
```

---

## defineExpose — expose child internals to parent

```ts
// Child
defineExpose({ focus, reset })

// Parent
const cardRef = ref()
cardRef.value.focus()   // call after onMounted
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

## Composables (Custom Hooks)

```ts
// src/composables/useCardEditor.ts
import { ref } from 'vue'

export function useCardEditor() {
  const isEditing = ref(false)
  const editText = ref('')

  function startEdit(title: string) {
    editText.value = title
    isEditing.value = true
  }

  return { isEditing, editText, startEdit }
}

// in any component
const { isEditing, editText, startEdit } = useCardEditor()
```

---

## provide / inject (Dependency Injection)

```ts
// Provider — any ancestor component or main.ts
import { provide, ref } from 'vue'
const theme = ref('dark')
provide('theme', theme)

// Consumer — any descendant, any depth
import { inject } from 'vue'
const theme = inject('theme')   // no prop drilling

// Use Symbol keys to avoid collision in large apps
export const THEME_KEY = Symbol('theme')
provide(THEME_KEY, theme)
const theme = inject(THEME_KEY)
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

```ts
const views = { board: BoardView, settings: SettingsView }
const active = ref('board')
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

## Suspense

```vue
<Suspense>
  <template #default><AsyncBoard /></template>
  <template #fallback><Spinner /></template>
</Suspense>

<!-- AsyncBoard.vue — top level await -->
<script setup>
const data = await fetch('/api/boards').then(r => r.json())
</script>
```

---

## Reactivity Utilities

```ts
import { markRaw, toRaw, shallowRef } from 'vue'

markRaw(obj)          // never proxy this — maps, charts, class instances
toRaw(reactiveObj)    // unwrap proxy → plain JS object for serialization
shallowRef([])        // only track .value replacement, not deep changes
```

---

## nextTick — wait for DOM update

```ts
import { nextTick } from 'vue'

cards.value.push(newCard)
await nextTick()           // DOM updated
inputEl.value?.focus()     // safe to access new element now
```

---

## defineOptions

```ts
defineOptions({
  name: 'KanbanCard',
  inheritAttrs: false,   // don't auto-apply parent attrs to root element
})
// forward attrs manually to specific element
// <input v-bind="$attrs" />
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

router.afterEach((to) => {
  document.title = to.meta.title ?? 'App'
})
```

```ts
// in component
import { useRoute, useRouter, onBeforeRouteLeave } from 'vue-router'

const route = useRoute()
route.params.id
route.meta.requiresAuth

const router = useRouter()
router.push('/board/1')
router.push({ name: 'board', params: { id: 1 } })
router.replace('/login')
router.back()

onBeforeRouteLeave(() => {
  if (unsaved.value) return false   // block navigation
})
```

---

## Pinia

```ts
// stores/board.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useBoardStore = defineStore('board', () => {
  // state
  const columns = ref([])

  // getter — cached computed
  const totalCards = computed(() =>
    columns.value.reduce((n, col) => n + col.cards.length, 0)
  )

  // action
  function addCard(columnId: number, title: string) {
    const col = columns.value.find(c => c.id === columnId)
    col?.cards.push({ id: Date.now(), title })
  }

  function $reset() { columns.value = [] }

  return { columns, totalCards, addCard, $reset }
}, {
  persist: { pick: ['columns'] }   // requires pinia-plugin-persistedstate
})
```

```ts
// in component
import { storeToRefs } from 'pinia'

const store = useBoardStore()
const { columns, totalCards } = storeToRefs(store)   // reactive destructure
const { addCard } = store                             // actions — no storeToRefs needed

store.$patch({ columns: [] })
store.$subscribe((mutation, state) => {})
store.$onAction(({ name, after }) => {})
store.$dispose()
```

```ts
// Pinia plugin — main.ts
pinia.use(({ store }) => {
  store.$api = { get: url => fetch(url).then(r => r.json()) }
})

// pinia.state.value — snapshot all stores
const snapshot = JSON.stringify(pinia.state.value)
pinia.state.value = JSON.parse(snapshot)   // restore
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
