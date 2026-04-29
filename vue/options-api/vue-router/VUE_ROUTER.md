# Vue Router — Options API

---

## Setup

```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),   // HTML5 mode (no hash)
  routes: []
})

export default router
```

```ts
// main.ts
app.use(router)
```

---

## Route Definitions

```ts
const routes = [
  // static route
  { path: '/', component: HomeView },

  // named route
  { path: '/about', name: 'about', component: AboutView },

  // dynamic param
  { path: '/board/:id', name: 'board', component: BoardView },

  // multiple params
  { path: '/user/:id/post/:postId', component: PostView },

  // lazy load (code split)
  {
    path: '/settings',
    component: () => import('../views/SettingsView.vue')
  },

  // nested routes
  {
    path: '/',
    component: AppLayout,
    children: [
      { path: '', component: HomeView },
      { path: 'dashboard', component: DashboardView }
    ]
  },

  // route meta
  {
    path: '/admin',
    component: AdminView,
    meta: { requiresAuth: true, role: 'admin' }
  },

  // redirect
  { path: '/old-path', redirect: '/new-path' },
  { path: '/old', redirect: { name: 'home' } },

  // catch-all 404
  { path: '/:pathMatch(.*)*', name: 'NotFound', component: NotFoundView }
]
```

---

## Accessing Route in Component

```js
export default {
  computed: {
    boardId() {
      return this.$route.params.id       // route param
    },
    page() {
      return this.$route.query.page      // query string ?page=2
    }
  },

  mounted() {
    console.log(this.$route.path)        // '/board/42'
    console.log(this.$route.name)        // 'board'
    console.log(this.$route.params)      // { id: '42' }
    console.log(this.$route.query)       // { page: '1' }
    console.log(this.$route.meta)        // { requiresAuth: true }
    console.log(this.$route.hash)        // '#section'
    console.log(this.$route.fullPath)    // '/board/42?page=1#section'
  }
}
```

---

## Programmatic Navigation

```js
export default {
  methods: {
    goHome() {
      this.$router.push('/')
    },

    goToBoard(id) {
      this.$router.push({ name: 'board', params: { id } })
    },

    goWithQuery() {
      this.$router.push({ path: '/search', query: { q: 'vue' } })
    },

    replaceRoute() {
      this.$router.replace('/login')   // no history entry
    },

    goBack() {
      this.$router.back()
    },

    goForward() {
      this.$router.forward()
    },

    goN(n) {
      this.$router.go(-2)   // go back 2 steps
    }
  }
}
```

---

## `<RouterLink>` — Declarative Navigation

```html
<RouterLink to="/">Home</RouterLink>
<RouterLink :to="{ name: 'board', params: { id: 1 } }">Board</RouterLink>
<RouterLink to="/about" active-class="active">About</RouterLink>

<!-- replace instead of push -->
<RouterLink to="/login" replace>Login</RouterLink>
```

---

## In-Component Route Guards

```js
export default {
  // called before route enters this component
  // NOTE: no access to `this` — use `next(vm => { ... })` instead
  beforeRouteEnter(to, from, next) {
    next(vm => {
      vm.fetchData(to.params.id)
    })
  },

  // called when route changes but component reuses (e.g. /user/1 → /user/2)
  beforeRouteUpdate(to, from) {
    this.fetchData(to.params.id)   // `this` available here
  },

  // called before leaving this route
  beforeRouteLeave(to, from) {
    if (this.hasUnsavedChanges) {
      return false   // cancel navigation
    }
  }
}
```

---

## Global Guards

```ts
// router/index.ts

// runs before every navigation
router.beforeEach((to, from) => {
  const auth = useAuthStore()

  if (to.meta.requiresAuth && !auth.isLoggedIn) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }
})

// runs after every navigation (no ability to cancel)
router.afterEach((to, from) => {
  document.title = to.meta.title ?? 'App'
})

// runs before route is resolved (after in-component guards)
router.beforeResolve(async (to) => {
  if (to.meta.requiresCamera) {
    await askForCameraPermission()
  }
})
```

---

## Scroll Behavior

```ts
const router = createRouter({
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) return savedPosition        // back/forward button
    if (to.hash) return { el: to.hash }            // anchor link
    return { top: 0 }                              // scroll to top
  }
})
```

---

## `<RouterView>` — Render Matched Component

```html
<!-- renders whatever route matches -->
<RouterView />

<!-- with transition -->
<RouterView v-slot="{ Component }">
  <Transition name="fade">
    <component :is="Component" />
  </Transition>
</RouterView>
```

---

## Quick Reference

| Task | Code |
|------|------|
| Current params | `this.$route.params.id` |
| Current query | `this.$route.query.page` |
| Navigate | `this.$router.push({ name, params })` |
| Replace | `this.$router.replace(path)` |
| Go back | `this.$router.back()` |
| Guard: leave check | `beforeRouteLeave` → `return false` |
| Guard: param change | `beforeRouteUpdate` |
| Global auth guard | `router.beforeEach` |
