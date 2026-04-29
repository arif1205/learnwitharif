# Vue 3 Slots — Options API

---

## Default Slot

Parent passes content, child renders it via `<slot />`.

```vue
<!-- Parent -->
<BaseCard>
  <p>This content goes into the slot</p>
</BaseCard>
```

```vue
<!-- BaseCard.vue -->
<template>
  <div class="card">
    <slot />   <!-- parent content renders here -->
  </div>
</template>
```

### Fallback Content

Rendered when parent provides nothing.

```vue
<!-- BaseCard.vue -->
<template>
  <div class="card">
    <slot>No content provided</slot>   <!-- fallback -->
  </div>
</template>
```

---

## Named Slots

Multiple slots in one component, each with a name.

```vue
<!-- Parent -->
<PageLayout>
  <template #header>
    <h1>Page Title</h1>
  </template>

  <template #default>
    <p>Main body content</p>
  </template>

  <template #footer>
    <p>Footer text</p>
  </template>
</PageLayout>
```

```vue
<!-- PageLayout.vue -->
<template>
  <div>
    <header><slot name="header" /></header>
    <main><slot /></main>           <!-- default slot -->
    <footer><slot name="footer" /></footer>
  </div>
</template>
```

> `#header` is shorthand for `v-slot:header`.

---

## Scoped Slots

Child passes data **up** to parent's slot template.

```vue
<!-- Child: DataTable.vue -->
<template>
  <table>
    <tr v-for="row in rows" :key="row.id">
      <slot :row="row" :index="i" />   <!-- pass data to slot -->
    </tr>
  </table>
</template>

<script>
export default {
  props: ['rows']
}
</script>
```

```vue
<!-- Parent — receives slot data -->
<DataTable :rows="users">
  <template #default="{ row, index }">
    <td>{{ index + 1 }}</td>
    <td>{{ row.name }}</td>
    <td>{{ row.email }}</td>
  </template>
</DataTable>
```

---

## Scoped Slots — Named

```vue
<!-- Child -->
<template>
  <ul>
    <li v-for="item in items" :key="item.id">
      <slot name="item" :item="item" :isActive="item.id === activeId" />
    </li>
  </ul>
</template>
```

```vue
<!-- Parent -->
<MyList :items="tasks">
  <template #item="{ item, isActive }">
    <span :class="{ bold: isActive }">{{ item.title }}</span>
  </template>
</MyList>
```

---

## Accessing Slots in Component

```js
export default {
  mounted() {
    // check if slot has content
    const hasHeader = !!this.$slots.header
    const hasDefault = !!this.$slots.default
  }
}
```

---

## Dynamic Slot Names

```html
<template v-slot:[dynamicSlotName]>
  Content
</template>
```

```js
data() {
  return { dynamicSlotName: 'header' }
}
```

---

## Quick Reference

| Pattern | Syntax |
|---------|--------|
| Default slot | `<slot />` / `<Component>content</Component>` |
| Named slot (define) | `<slot name="header" />` |
| Named slot (use) | `<template #header>...</template>` |
| Scoped slot (pass data) | `<slot :item="item" />` |
| Scoped slot (receive) | `<template #default="{ item }">` |
| Fallback content | `<slot>fallback text</slot>` |
| Check slot exists | `this.$slots.slotName` |
