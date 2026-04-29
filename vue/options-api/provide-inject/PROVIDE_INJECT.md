# Vue 3 provide / inject — Options API

---

## Basic Usage

Pass data down component tree without prop drilling.

```js
// Ancestor component (any level up)
export default {
  provide() {
    return {
      appTitle: 'My App',
      maxItems: 10
    }
  }
}
```

```js
// Descendant component (any level down)
export default {
  inject: ['appTitle', 'maxItems'],

  mounted() {
    console.log(this.appTitle)   // 'My App'
    console.log(this.maxItems)   // 10
  }
}
```

---

## Inject with Default Value

```js
export default {
  inject: {
    theme: {
      default: 'light'   // fallback if no provider found
    },
    appTitle: {
      default: 'Default App'
    }
  }
}
```

---

## Reactive provide

By default, provided values are **not reactive**. Use `computed` to make them reactive.

```js
import { computed } from 'vue'

// Provider
export default {
  data() {
    return { theme: 'dark' }
  },

  provide() {
    return {
      // computed = reactive — consumer sees updates when this.theme changes
      theme: computed(() => this.theme)
    }
  }
}
```

```js
// Consumer — theme.value (it's a computed ref)
export default {
  inject: ['theme'],

  computed: {
    currentTheme() {
      return this.theme.value   // unwrap the computed ref
    }
  }
}
```

---

## App-Level provide

Provide to every component in the app.

```ts
// main.ts
app.provide('apiUrl', 'https://api.example.com')
app.provide('version', '1.0.0')
```

```js
// any component
export default {
  inject: ['apiUrl', 'version']
}
```

---

## Symbol Keys (Avoid Collisions)

Use Symbol keys in large apps to prevent name collisions between plugins/libraries.

```ts
// keys.ts — shared symbols
export const THEME_KEY = Symbol('theme')
export const API_KEY = Symbol('api')
```

```js
// Provider
import { THEME_KEY } from '@/keys'

export default {
  provide() {
    return {
      [THEME_KEY]: 'dark'
    }
  }
}
```

```js
// Consumer
import { THEME_KEY } from '@/keys'

export default {
  inject: {
    theme: { from: THEME_KEY, default: 'light' }
  }
}
```

---

## provide vs props

| | `props` | `provide/inject` |
|---|---------|-----------------|
| Scope | Direct parent → child | Any ancestor → any descendant |
| Reactivity | Reactive | Not by default (use `computed`) |
| Use when | Normal parent-child | Global config, themes, services |
| Coupling | Explicit | Implicit — harder to trace |
