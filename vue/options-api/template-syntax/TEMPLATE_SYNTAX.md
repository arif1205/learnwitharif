# Template Syntax — Options API

Vue templates are compiled to render functions. Every directive (`:`, `@`, `v-if`, etc.) is syntactic sugar — not magic, just compiler macros. The compiler transforms `v-for` with `:key` into keyed VDOM patches; without `:key`, Vue reuses nodes by position, causing subtle bugs with stateful elements.

---

## Interpolation

```html
{{ card.title }}
{{ card.priority.toUpperCase() }}
{{ isOverLimit ? 'Column full' : card.title }}
```

Only single expressions. No statements (`if`, `for`, variable declarations). For complex logic, use a computed.

`v-html` renders raw HTML — XSS risk. Only use with sanitized/trusted content.

```html
<div v-html="card.descriptionHtml"></div>
```

---

## Attribute Binding (`v-bind` / `:`)

```html
<!-- scalar -->
<input :placeholder="column.title" :disabled="readonly" />

<!-- dynamic attribute name -->
<div :[attrName]="attrValue"></div>

<!-- spread object -->
<div v-bind="cardAttributes"></div>

<!-- boolean — only renders attr if truthy -->
<button :disabled="column.cards.length >= maxCards">Add Card</button>
```

---

## Class Binding

```html
<!-- object syntax — key = class name, value = condition -->
<div :class="{ 'card--high': card.priority === 'high', 'card--dragging': isDragging }">

<!-- array syntax -->
<div :class="[priorityClass, isSelected ? 'card--selected' : '']">

<!-- combined with static class -->
<div class="card" :class="{ 'card--focused': isFocused }">
```

```js
computed: {
  priorityClass(): string { return `card--${this.card.priority}` }
}
```

---

## Style Binding

```html
<!-- object -->
<div :style="{ backgroundColor: column.color, height: columnHeight + 'px' }">

<!-- array — merges multiple style objects -->
<div :style="[baseStyle, overrideStyle]">
```

Vue automatically adds vendor prefixes for CSS properties that need them.

---

## Conditional Rendering

```html
<div v-if="board.columns.length === 0">
  <EmptyBoardState />
</div>
<div v-else-if="loading">
  <BoardSkeleton />
</div>
<div v-else>
  <!-- board content -->
</div>
```

`v-show` keeps the element in the DOM — only toggles `display: none`. Use when the element toggles frequently.

```html
<CardActions v-show="isHovered" />
```

**`v-if` with `v-for`:** don't put them on the same element. `v-if` has higher priority and can't access `v-for` variables. Use a wrapping `<template>`.

```html
<!-- BAD -->
<li v-for="card in cards" v-if="card.visible" :key="card.id">

<!-- GOOD -->
<template v-for="card in cards" :key="card.id">
  <li v-if="card.visible">{{ card.title }}</li>
</template>
```

---

## List Rendering (`v-for`)

```html
<!-- array -->
<KanbanColumn
  v-for="col in columns"
  :key="col.id"
  :column="col"
  @add-card="addCard"
/>

<!-- array with index -->
<div v-for="(col, index) in columns" :key="col.id">
  {{ index + 1 }}. {{ col.title }}
</div>

<!-- object properties -->
<div v-for="(value, key, index) in card" :key="key">
  {{ key }}: {{ value }}
</div>

<!-- range -->
<div v-for="n in 5" :key="n">{{ n }}</div>
```

**`:key` rules:**
- Always use `:key`. Without it, Vue reuses DOM nodes by position — inputs retain their values when list reorders.
- Use stable unique IDs, not array indexes (index-based keys break animations and input state on insert/delete).

---

## Event Handling (`v-on` / `@`)

```html
<!-- method reference — Vue passes $event automatically -->
<button @click="deleteCard">Delete</button>

<!-- inline with args -->
<button @click="moveCard(card.id, targetColumnId)">Move</button>

<!-- access $event in inline -->
<input @keydown="handleKey($event, card.id)" />

<!-- multiple handlers -->
<button @click="saveCard(); closeModal()">Save</button>
```

### Event Modifiers

```html
@click.stop          <!-- stopPropagation — prevents card click when clicking action button -->
@click.prevent       <!-- preventDefault — useful for form submits -->
@click.self          <!-- only fires if click target IS this element, not a child -->
@click.once          <!-- unregisters after first fire -->
@mousedown.left      <!-- only left button -->
@keydown.enter       <!-- enter key only -->
@keydown.ctrl.enter  <!-- ctrl+enter -->
@keydown.escape
```

**`.stop` vs `.self`:** `.stop` blocks propagation upward. `.self` fires only when the target is the element itself — doesn't affect propagation.

---

## Two-Way Binding (`v-model`)

```html
<!-- text input -->
<input v-model="card.title" />

<!-- trimmed, number-cast -->
<input v-model.trim="searchQuery" />
<input v-model.number="maxCards" type="number" />

<!-- lazy — syncs on change event, not input -->
<textarea v-model.lazy="card.description" />

<!-- checkbox — boolean -->
<input type="checkbox" v-model="card.done" />

<!-- checkbox — array of values -->
<input type="checkbox" v-model="selectedCardIds" :value="card.id" />

<!-- radio -->
<input type="radio" v-model="card.priority" value="high" />
<input type="radio" v-model="card.priority" value="medium" />

<!-- select -->
<select v-model="card.assignee">
  <option v-for="user in users" :key="user.id" :value="user.id">
    {{ user.name }}
  </option>
</select>
```

---

## Template Refs

Get a reference to a DOM element or child component instance.

```html
<div ref="columnList" class="columns">
  <KanbanCard v-for="card in cards" :key="card.id" :ref="el => registerCardRef(el, card.id)" />
</div>
```

```js
export default {
  mounted() {
    const el = this.$refs.columnList as HTMLElement
    el.scrollTo({ left: el.scrollWidth, behavior: 'smooth' })
  },

  methods: {
    registerCardRef(el: Element | null, cardId: number) {
      if (el) this.cardRefs.set(cardId, el as HTMLElement)
      else this.cardRefs.delete(cardId)
    }
  }
}
```

---

## Special Elements

```html
<!-- <template> — logical grouping without DOM output -->
<template v-if="hasPermission">
  <AddCardButton />
  <ColumnSettingsButton />
</template>

<!-- <component> — dynamic component -->
<component :is="currentPanelComponent" v-bind="panelProps" />

<!-- :key — force re-mount when key changes -->
<BoardView :key="$route.params.boardId" />
```

---

## Expressions vs Statements

Templates only allow expressions (things that return a value), not statements.

```html
{{ card.title }}                    ✓ expression
{{ card.priority === 'high' ? '🔴' : '' }}   ✓ ternary
{{ cards.filter(c => !c.done).length }}      ✓ method chain

{{ let x = 1 }}                     ✗ declaration
{{ if (x) {} }}                     ✗ if statement

<!-- for complex logic, use computed instead -->
```
