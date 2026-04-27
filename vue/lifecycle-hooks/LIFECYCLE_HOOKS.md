# Vue 3 Lifecycle Hooks

---

## Hook Execution Order

```
setup()
  │
  ├── onBeforeMount()
  ├── onMounted()          ← DOM ready
  │
  ├── [reactive data changes]
  │     ├── onBeforeUpdate()
  │     └── onUpdated()
  │
  ├── [component removed]
  │     ├── onBeforeUnmount()
  │     └── onUnmounted()   ← cleanup here
  │
  ├── onErrorCaptured()    ← child errors
  ├── onActivated()        ← <KeepAlive> enter
  └── onDeactivated()      ← <KeepAlive> leave
```

---

## 1. `onBeforeMount`

Runs before DOM is created. Reactive data ready, but `$el` not yet in DOM.

```vue
<script setup>
import { onBeforeMount } from 'vue'

onBeforeMount(() => {
  console.log('Before mount — DOM not ready yet')
})
</script>
```

**Use for:** last-chance data prep before render. Rarely needed.

---

## 2. `onMounted`

DOM is fully created and inserted. Safe to access DOM elements.

```vue
<script setup>
import { ref, onMounted } from 'vue'

const inputRef = ref(null)

onMounted(() => {
  inputRef.value.focus()        // direct DOM access
  fetchInitialData()            // API call on load
})
</script>

<template>
  <input ref="inputRef" />
</template>
```

**Use for:** DOM access, API calls, third-party library init (charts, maps).

---

## 3. `onBeforeUpdate`

Runs when reactive data changes, before DOM re-renders.

```vue
<script setup>
import { ref, onBeforeUpdate } from 'vue'

const count = ref(0)

onBeforeUpdate(() => {
  console.log('About to update. Current DOM still shows old value.')
})
</script>

<template>
  <button @click="count++">{{ count }}</button>
</template>
```

**Use for:** snapshot DOM state before update (e.g., scroll position).

---

## 4. `onUpdated`

DOM has re-rendered after reactive data changed.

```vue
<script setup>
import { ref, onUpdated } from 'vue'

const list = ref(['a', 'b'])

onUpdated(() => {
  console.log('DOM updated, list length:', list.value.length)
})
</script>
```

> **Warning:** Don't mutate reactive state here — causes infinite loop.

**Use for:** post-render DOM reads, syncing third-party widgets.

---

## 5. `onBeforeUnmount`

Component still fully functional, about to be destroyed.

```vue
<script setup>
import { onBeforeUnmount } from 'vue'

const timer = setInterval(() => console.log('tick'), 1000)

onBeforeUnmount(() => {
  clearInterval(timer)   // cleanup before destroy
})
</script>
```

**Use for:** final cleanup while component APIs still work.

---

## 6. `onUnmounted`

Component fully removed from DOM. Refs, child components gone.

```vue
<script setup>
import { onMounted, onUnmounted } from 'vue'

function handleResize() {
  console.log('window width:', window.innerWidth)
}

onMounted(() => window.addEventListener('resize', handleResize))
onUnmounted(() => window.removeEventListener('resize', handleResize))
</script>
```

**Use for:** remove event listeners, cancel subscriptions, disconnect WebSockets.

---

## 7. `onErrorCaptured`

Catches errors from any descendant component.

```vue
<script setup>
import { ref, onErrorCaptured } from 'vue'

const error = ref(null)

onErrorCaptured((err, instance, info) => {
  error.value = err.message
  console.error('Caught from child:', err, info)
  return false   // false = stop error propagation
})
</script>

<template>
  <div v-if="error">Error: {{ error }}</div>
  <ChildComponent v-else />
</template>
```

**Use for:** error boundaries around risky child trees.

---

## 8. `onActivated` / `onDeactivated`

Only fires for components wrapped in `<KeepAlive>`.

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
<script setup>
import { onActivated, onDeactivated } from 'vue'

onActivated(() => {
  console.log('Tab shown — resume polling')
  startPolling()
})

onDeactivated(() => {
  console.log('Tab hidden — pause polling')
  stopPolling()
})
</script>
```

**Use for:** pause/resume side effects when tab switches without destroying component.

---

## 9. `onRenderTracked` / `onRenderTriggered` (Dev Only)

Debug which reactive dependency triggered a re-render.

```vue
<script setup>
import { onRenderTracked, onRenderTriggered } from 'vue'

onRenderTracked((event) => {
  console.log('Dependency tracked:', event)
})

onRenderTriggered((event) => {
  console.log('Re-render triggered by:', event)
  // event.key, event.target, event.type
})
</script>
```

**Use for:** debugging unexpected re-renders. Stripped in production builds.

---

## Real-World Pattern: Data Fetch + Cleanup

```vue
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const data = ref(null)
const loading = ref(true)
let controller = null

onMounted(async () => {
  controller = new AbortController()
  try {
    const res = await fetch('/api/data', { signal: controller.signal })
    data.value = await res.json()
  } catch (e) {
    if (e.name !== 'AbortError') console.error(e)
  } finally {
    loading.value = false
  }
})

onUnmounted(() => {
  controller?.abort()   // cancel in-flight request if component removed
})
</script>
```

---

## Quick Reference

| Hook | When | Common Use |
|------|------|------------|
| `onBeforeMount` | Before DOM created | Last data prep |
| `onMounted` | DOM ready | API calls, DOM access, lib init |
| `onBeforeUpdate` | Before re-render | Snapshot DOM |
| `onUpdated` | After re-render | Post-render DOM reads |
| `onBeforeUnmount` | Before destroy | Cleanup while APIs live |
| `onUnmounted` | After destroy | Remove listeners, cancel subs |
| `onErrorCaptured` | Child error | Error boundaries |
| `onActivated` | KeepAlive shown | Resume effects |
| `onDeactivated` | KeepAlive hidden | Pause effects |
| `onRenderTracked` | Dev: dep tracked | Debug reactivity |
| `onRenderTriggered` | Dev: re-render | Debug re-renders |
