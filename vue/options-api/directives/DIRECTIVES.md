# Vue 3 Directives — Options API

---

## Built-in Directives (Quick Reference)

| Directive | Purpose |
|-----------|---------|
| `v-if` / `v-else` / `v-else-if` | Conditional render |
| `v-show` | Toggle `display:none` |
| `v-for` | List rendering |
| `v-bind` / `:` | Bind attribute |
| `v-on` / `@` | Event listener |
| `v-model` | Two-way binding |
| `v-once` | Render once |
| `v-pre` | Skip compilation |
| `v-html` | Render raw HTML |
| `v-memo` | Memoize subtree |

See [template-syntax/TEMPLATE_SYNTAX.md](../template-syntax/TEMPLATE_SYNTAX.md) for full usage.

---

## Custom Directives — Global

```ts
// main.ts
app.directive('focus', {
  mounted(el) {
    el.focus()
  }
})
```

```html
<input v-focus />
```

---

## Custom Directives — Local

```js
export default {
  directives: {
    focus: {
      mounted(el) {
        el.focus()
      }
    }
  }
}
```

```html
<input v-focus />
```

---

## Directive Hooks

```js
app.directive('my-directive', {
  // element created, before attrs/children applied
  created(el, binding, vnode) {},

  // before element inserted into DOM
  beforeMount(el, binding, vnode) {},

  // after element inserted into DOM
  mounted(el, binding, vnode) {},

  // before parent component updates
  beforeUpdate(el, binding, vnode, prevVnode) {},

  // after parent component and children updated
  updated(el, binding, vnode, prevVnode) {},

  // before parent component unmounts
  beforeUnmount(el, binding, vnode) {},

  // after parent component unmounts
  unmounted(el, binding, vnode) {}
})
```

---

## Directive Binding Object

```js
app.directive('example', {
  mounted(el, binding) {
    binding.value       // value passed to directive: v-example="value"
    binding.oldValue    // previous value (beforeUpdate / updated only)
    binding.arg         // argument: v-example:arg
    binding.modifiers   // { mod1: true, mod2: true }: v-example.mod1.mod2
    binding.instance    // component instance
    binding.dir         // directive definition object
  }
})
```

---

## Real Examples

### `v-focus` — Auto Focus

```js
app.directive('focus', {
  mounted(el) { el.focus() }
})
```

```html
<input v-focus />
```

---

### `v-click-outside` — Detect Outside Click

```js
app.directive('click-outside', {
  mounted(el, binding) {
    el._clickOutsideHandler = (event) => {
      if (!el.contains(event.target)) {
        binding.value(event)   // call the passed function
      }
    }
    document.addEventListener('click', el._clickOutsideHandler)
  },
  unmounted(el) {
    document.removeEventListener('click', el._clickOutsideHandler)
  }
})
```

```html
<div v-click-outside="closeDropdown">
  <button @click="open = !open">Toggle</button>
  <ul v-if="open">...</ul>
</div>
```

---

### `v-tooltip` — Simple Tooltip

```js
app.directive('tooltip', {
  mounted(el, binding) {
    el.title = binding.value
    el.style.cursor = 'help'
  },
  updated(el, binding) {
    el.title = binding.value
  }
})
```

```html
<button v-tooltip="'Save your changes'">Save</button>
```

---

### `v-color` — Dynamic Color with Argument

```js
app.directive('color', {
  mounted(el, binding) {
    // v-color:background="'red'" or v-color:color="'blue'"
    el.style[binding.arg ?? 'color'] = binding.value
  }
})
```

```html
<p v-color="'red'">Red text</p>
<p v-color:background="'yellow'">Yellow bg</p>
```

---

## Shorthand (mounted + updated only)

When you only need `mounted` and `updated` with same behavior:

```js
app.directive('color', (el, binding) => {
  el.style.color = binding.value   // runs on mounted and updated
})
```
