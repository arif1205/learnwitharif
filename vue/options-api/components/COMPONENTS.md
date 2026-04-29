# Vue 3 Built-in Components — Options API

---

## `<Teleport>` — Render Outside Component Tree

Moves slot content to a different DOM node while keeping component logic in place.

```vue
<template>
  <button @click="showModal = true">Open Modal</button>

  <!-- renders inside <body>, not inside current component -->
  <Teleport to="body">
    <div v-if="showModal" class="modal-overlay">
      <div class="modal">
        <p>Modal content</p>
        <button @click="showModal = false">Close</button>
      </div>
    </div>
  </Teleport>
</template>

<script>
export default {
  data() {
    return { showModal: false }
  }
}
</script>
```

**Use for:** modals, toasts, tooltips — elements that need to escape `overflow:hidden` or `z-index` stacking contexts.

`to` accepts any CSS selector: `"body"`, `"#app"`, `".portal"`.

---

## `<KeepAlive>` — Cache Component State

Wraps dynamic components to preserve state when they're hidden instead of destroying them.

```vue
<template>
  <KeepAlive>
    <component :is="currentTab" />
  </KeepAlive>
</template>
```

Without `<KeepAlive>`: switching tabs destroys and recreates the component — state lost.
With `<KeepAlive>`: component is cached — state preserved, `activated`/`deactivated` hooks fire.

### Include / Exclude

```html
<!-- only cache these components -->
<KeepAlive include="HomeView,DashboardView">
  <RouterView />
</KeepAlive>

<!-- cache everything except these -->
<KeepAlive exclude="LoginView">
  <RouterView />
</KeepAlive>

<!-- limit cache size (oldest evicted first) -->
<KeepAlive :max="5">
  <RouterView />
</KeepAlive>
```

### Lifecycle with KeepAlive

```js
export default {
  activated() {
    // component entered view (from cache)
    this.startPolling()
  },
  deactivated() {
    // component left view (moved to cache)
    this.stopPolling()
  }
}
```

---

## `<Transition>` — Animate Single Element

```vue
<template>
  <button @click="show = !show">Toggle</button>

  <Transition name="fade">
    <p v-if="show">Hello</p>
  </Transition>
</template>

<style>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
```

### CSS Classes Applied

```
enter:   [name]-enter-from  →  [name]-enter-active  →  [name]-enter-to
leave:   [name]-leave-from  →  [name]-leave-active  →  [name]-leave-to
```

### Transition Modes

```html
<!-- wait for old to leave before new enters -->
<Transition name="fade" mode="out-in">
  <component :is="currentView" :key="currentView" />
</Transition>
```

---

## `<TransitionGroup>` — Animate Lists

```vue
<template>
  <TransitionGroup name="list" tag="ul">
    <li v-for="item in items" :key="item.id">
      {{ item.text }}
    </li>
  </TransitionGroup>
</template>

<style>
.list-enter-active,
.list-leave-active {
  transition: all 0.3s ease;
}
.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateX(-20px);
}
/* animate existing items moving to fill gap */
.list-move {
  transition: transform 0.3s ease;
}
</style>
```

---

## Dynamic Component

Switch between components at runtime using `:is`.

```vue
<template>
  <div>
    <button @click="tab = 'HomeTab'">Home</button>
    <button @click="tab = 'ProfileTab'">Profile</button>

    <component :is="currentComponent" />
  </div>
</template>

<script>
import HomeTab from './HomeTab.vue'
import ProfileTab from './ProfileTab.vue'

export default {
  components: { HomeTab, ProfileTab },

  data() {
    return { tab: 'HomeTab' }
  },

  computed: {
    currentComponent() {
      return this.tab   // matches registered component name
    }
  }
}
</script>
```

With `<KeepAlive>`:

```html
<KeepAlive>
  <component :is="currentComponent" />
</KeepAlive>
```

---

## Async Component

Load component only when needed (code splitting).

```js
import { defineAsyncComponent } from 'vue'

export default {
  components: {
    // basic
    HeavyChart: defineAsyncComponent(
      () => import('./HeavyChart.vue')
    ),

    // with options
    DataTable: defineAsyncComponent({
      loader: () => import('./DataTable.vue'),
      loadingComponent: LoadingSpinner,
      errorComponent: ErrorDisplay,
      delay: 200,        // show loading after 200ms
      timeout: 5000      // show error after 5s
    })
  }
}
```

---

## `<Suspense>` — Async Loading Boundary

Coordinate loading states for async components.

```vue
<template>
  <Suspense>
    <template #default>
      <AsyncDataComponent />
    </template>
    <template #fallback>
      <LoadingSpinner />
    </template>
  </Suspense>
</template>
```

> **Note:** `<Suspense>` is experimental. In Options API, pair with `defineAsyncComponent` since top-level `await` is a `<script setup>` feature.
