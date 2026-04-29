# Vue 3 Cheat Sheet — Options API

Overview of all features. Each section links to a dedicated file with full examples.

---

## Project Boot Sequence

```
index.html → <div id="app"> → main.ts → createApp(App).mount('#app')
```

```ts
// main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import router from './router'
import App from './App.vue'

const app = createApp(App)
app.use(createPinia())
app.use(router)
app.mount('#app')   // always last
```

---

## Single File Component (SFC)

```vue
<script>
export default {
  name: 'MyComponent',
  components: {},      // local component registration
  props: {},           // incoming data from parent
  emits: [],           // events this component fires
  data() { return {} },
  computed: {},
  watch: {},
  methods: {},
  mounted() {}         // lifecycle hook
}
</script>

<template><!-- HTML + Vue syntax --></template>

<style scoped>/* CSS scoped to this component */</style>
```

---

## Reactivity

State lives in `data()`. `computed` for derived values. `watch` for side effects. `methods` for handlers.

```js
export default {
  data() { return { count: 0 } },
  computed: {
    doubled() { return this.count * 2 }
  },
  watch: {
    count(newVal) { console.log(newVal) }
  },
  methods: {
    increment() { this.count++ }
  }
}
```

**Full reference:** [reactivity/REACTIVITY.md](reactivity/REACTIVITY.md)

---

## Template Syntax

Interpolation, `v-if`, `v-for`, `v-model`, event/key/class/style binding.

```html
{{ message }}
:attr="value"           v-if="cond"        v-for="item in list" :key="item.id"
@click="handler"        v-model="value"    @click.prevent  @keyup.enter
:class="{ active: isActive }"              :style="{ color: textColor }"
```

**Full reference:** [template-syntax/TEMPLATE_SYNTAX.md](template-syntax/TEMPLATE_SYNTAX.md)

---

## Props & Emits

Parent passes data in via `props`. Child sends events up via `$emit`.

```js
export default {
  props: { title: { type: String, required: true }, count: { type: Number, default: 0 } },
  emits: ['submit'],
  methods: {
    handleSubmit() { this.$emit('submit', this.formData) }
  }
}
```

**Full reference:** [props-emits/PROPS_EMITS.md](props-emits/PROPS_EMITS.md)

---

## Lifecycle Hooks

Hook into component creation, DOM mounting, updates, and destruction.

```js
export default {
  beforeCreate() {},   created() {},
  beforeMount() {},    mounted() {},       // DOM ready here
  beforeUpdate() {},   updated() {},
  beforeUnmount() {},  unmounted() {},     // cleanup here
  activated() {},      deactivated() {},   // KeepAlive
  errorCaptured() {},  serverPrefetch() {} // SSR
}
```

**Full reference:** [lifecycle-hooks/LIFECYCLE_HOOKS.md](lifecycle-hooks/LIFECYCLE_HOOKS.md)

---

## Slots

Pass template content from parent into child component.

```html
<!-- default -->         <BaseCard><p>Content</p></BaseCard>
<!-- named -->           <template #header><h1>Title</h1></template>
<!-- scoped -->          <template #item="{ row }">{{ row.name }}</template>
```

**Full reference:** [slots/SLOTS.md](slots/SLOTS.md)

---

## provide / inject

Pass data to deeply nested descendants without prop drilling.

```js
// ancestor
export default { provide() { return { theme: 'dark' } } }

// descendant (any depth)
export default { inject: ['theme'] }
```

**Full reference:** [provide-inject/PROVIDE_INJECT.md](provide-inject/PROVIDE_INJECT.md)

---

## Custom Directives

Extend template syntax with custom DOM behavior.

```js
app.directive('focus', { mounted(el) { el.focus() } })
// <input v-focus />
```

**Full reference:** [directives/DIRECTIVES.md](directives/DIRECTIVES.md)

---

## Built-in Components

`<Teleport>`, `<KeepAlive>`, `<Transition>`, `<TransitionGroup>`, dynamic & async components.

```html
<Teleport to="body"><Modal v-if="show" /></Teleport>
<KeepAlive :max="5"><component :is="currentTab" /></KeepAlive>
<Transition name="fade"><div v-if="show" /></Transition>
```

**Full reference:** [components/COMPONENTS.md](components/COMPONENTS.md)

---

## Vue Router

Client-side routing with navigation guards, dynamic params, and lazy loading.

```js
// in component
this.$route.params.id          // current route params
this.$router.push({ name: 'board', params: { id } })   // navigate
```

**Full reference:** [vue-router/VUE_ROUTER.md](vue-router/VUE_ROUTER.md)

---

## Pinia (State Management)

Global state store. Map to Options API via `mapStores`, `mapState`, `mapActions`.

```js
import { mapState, mapActions } from 'pinia'
export default {
  computed: { ...mapState(useCounterStore, ['count']) },
  methods:  { ...mapActions(useCounterStore, ['increment']) }
}
```

**Full reference:** [pinia/PINIA.md](pinia/PINIA.md)

---

## App Instance API

```ts
app.use(plugin)
app.component('Name', Component)
app.directive('name', hooks)
app.provide(key, value)
app.config.globalProperties.$x = value
app.config.errorHandler = (err, instance, info) => {}
app.unmount()
```
