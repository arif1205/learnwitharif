# WordPress Plugin Architecture — Master Index

Learning a real WordPress plugin (**FluentCommunity**) as a MERN/React engineer.
Each topic below = one short point + link to a full deep-dive file.

> Reference codebase: `fluent-community` (WPFluent framework, Vue 3 SPA).
> Legend: ✅ written · 🔲 planned

---

## Mental model (read first)

WordPress = a host runtime you cannot edit (like a big Express app). A plugin = a guest package that **hooks into** WP's lifecycle events. You never own `main()`. WP boots, fires events, your listeners run. Everything else is layered on top of that idea.

---

## 1. Entry point ✅
The `plugin-name.php` file. WP scans its header comment to discover the plugin, then runs it. Defines global constants, loads the Composer autoloader, and kicks off the bootstrap closure. It wires **nothing business-related** — just setup.
→ [ENTRY_FILE.md](ENTRY_FILE.md)

## 2. Bootstrap & Application 🔲
`boot/app.php` returns a closure that creates the `Application` (DI container), registers activation/deactivation hooks, schedules cron jobs (Action Scheduler), loads modules, and fires the plugin's own lifecycle events (`portal_loaded`, `on_wp_init`).
→ [BOOTSTRAP.md](BOOTSTRAP.md)

## 3. Hooks — actions & filters 🔲
The pub/sub core of WordPress. **action** = fire-and-forget event (`do_action`). **filter** = value transformer in a pipe (`add_filter`, must return). Custom hooks are namespaced `fluent_community/`. Learn this before anything else.
→ [HOOKS.md](HOOKS.md)

## 4. Hook handlers 🔲
`app/Hooks/actions.php` instantiates Handler classes and calls `->register()` on each. A handler binds its own methods to WP hooks — this is the plugin's "startup wiring", not logic.
→ [HOOK_HANDLERS.md](HOOK_HANDLERS.md)

## 5. Autoloading & namespaces (PSR-4) 🔲
Composer maps namespace → folder. `FluentCommunity\App\Models\Feed` = `app/Models/Feed.php`. Namespace path = file path. This is how you navigate PHP without an IDE jump.
→ [AUTOLOAD_PSR4.md](AUTOLOAD_PSR4.md)

## 6. WPFluent framework 🔲
A mini-Laravel bundled via Composer (`vendor/wpfluent/framework`). Gives you a Router, Request, DI container, and Eloquent-style ORM on top of raw WordPress. Maps closely to Express + Prisma.
→ [WPFLUENT.md](WPFLUENT.md)

## 7. REST routing 🔲
`app/Http/Routes/api.php`. Express-like: `$router->get('/', 'FeedsController@get')`. Grouped by prefix, guarded by a Policy. All routes live under REST base `fluent-community/v2`.
→ [ROUTING.md](ROUTING.md)

## 8. Controllers 🔲
`app/Http/Controllers/`. A method receives `$request`, sanitizes input, talks to models, returns an array (auto-JSON). Thin layer — orchestration, not domain rules.
→ [CONTROLLERS.md](CONTROLLERS.md)

## 9. Policies (authorization) 🔲
`app/Http/Policies/`. Middleware/guards. `verifyRequest()` runs before every route in a group; method-named funcs override per route. **Gotcha:** a destructive route with no matching policy method silently falls back to the weaker `verifyRequest()`.
→ [POLICIES.md](POLICIES.md)

## 10. Models & ORM 🔲
`app/Models/`. Eloquent-style, like Prisma models. `$table`, `$fillable`, `$guarded`, `$casts`, plus `boot()` lifecycle hooks (`creating`, `updating`). Chainable queries: `Feed::where(...)->get()`.
→ [MODELS.md](MODELS.md)

## 11. Database & migrations 🔲
Custom tables prefixed `fcom_`. Migrations in `database/`, run on version bump. **Multi-type tables:** one table (e.g. `fcom_spaces`) holds several models split by a `type` column via a global scope.
→ [DATABASE.md](DATABASE.md)

## 12. Input sanitization & security 🔲
Never trust raw input. `sanitize_text_field()`, `intval()`, `wp_kses_post()`, nonces. `$request->getSafe('field', 'sanitizer', default)`. Escape on output. A required WP discipline, not optional.
→ [SECURITY.md](SECURITY.md)

## 13. Modules (pluggable features) 🔲
`Modules/` — Auth, Course, Gutenberg, Integrations, etc. Each self-registers via hooks. The Pro plugin extends the free one purely through hooks — no direct calls.
→ [MODULES.md](MODULES.md)

## 14. Background jobs & cron 🔲
Action Scheduler (WooCommerce library). Recurring jobs (`..._hour_jobs`, `..._daily_jobs`) scheduled at activation. WP's own `wp_cron` is request-triggered, not real cron.
→ [CRON_JOBS.md](CRON_JOBS.md)

## 15. Frontend SPA 🔲
Vue 3 **Options API only** (`data/computed/methods/watch`), Pinia stores (Redux/Zustand), Element Plus UI, Vite build. Two separate SPAs: Portal (`/portal`) and Admin (wp-admin). REST via `$api` / `$get` / `$post`.
→ [FRONTEND.md](FRONTEND.md)

## 16. i18n (translation) 🔲
PHP: `__('text', 'fluent-community')`. Vue: `$t('text')`. A "text domain" scopes strings to the plugin. Strings extracted to translation files.
→ [I18N.md](I18N.md)

---

## Full request lifecycle (ties it together)

Vue store → REST call `fluent-community/v2/feeds` → Router matches → Policy authorizes → Controller sanitizes + orchestrates → Model writes to `fcom_posts` → fires `do_action('fluent_community/feed/created')` → handlers react (notify, email, log) → JSON back → Pinia updates → reactive UI.

→ [REQUEST_LIFECYCLE.md](REQUEST_LIFECYCLE.md) 🔲

---

## Suggested reading order
Entry point → Bootstrap → Hooks → Routing → Controllers → Models → Frontend.
Skip deep PHP/Laravel theory; absorb by reading a controller + its model side by side.
