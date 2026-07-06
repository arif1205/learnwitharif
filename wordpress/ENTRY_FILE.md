# Entry File — `plugin-name.php`

The main plugin file. In FluentCommunity it is `fluent-community.php` (name matches the plugin folder). This is the **only** file WordPress loads automatically; everything else is pulled in from here.

> MERN analogy: this file is `package.json` (the discovery manifest) and `index.js` (the boot entry) fused into one.

Full reference file below:

```php
<?php

defined('ABSPATH') or die;

/**
 * Plugin Name: FluentCommunity
 * Description: The super-fast Community Plugin for WordPress
 * Version: 2.6.1
 * Author: WPManageNinja LLC
 * Author URI: https://fluentcommunity.co
 * Plugin URI: https://fluentcommunity.co
 * License: GPLv2 or later
 * Text Domain: fluent-community
 * Domain Path: /language
 */

define('FLUENT_COMMUNITY_PLUGIN_VERSION', '2.6.1');
define('FLUENT_COMMUNITY_PLUGIN_DIR', plugin_dir_path(__FILE__));
define('FLUENT_COMMUNITY_PLUGIN_URL', plugin_dir_url(__FILE__));
define('FLUENT_COMMUNITY_DIR_FILE', __FILE__);
define('FLUENT_COMMUNITY_START_TIME', microtime(true));
define('FLUENT_COMMUNITY_DB_VERSION', '1.0.5');
define('FLUENT_COMMUNITY_MIN_PRO_VERSION', '2.6.0');

if (!defined('FLUENTCRM_COMMUNITY_UPLOAD_DIR')) {
    define('FLUENT_COMMUNITY_UPLOAD_DIR', 'fluent-community');
}

require __DIR__ . '/vendor/autoload.php';

call_user_func(function ($bootstrap) {
    $bootstrap(__FILE__);
}, require(__DIR__ . '/boot/app.php'));
```

Each part, top to bottom:

---

## 1. `<?php` — open tag
PHP files start in "HTML mode". The `<?php` tag switches to PHP code. A pure-PHP file (no HTML output) also **omits the closing `?>`** on purpose — a trailing newline after `?>` can break HTTP headers. You'll see this convention everywhere.

## 2. `defined('ABSPATH') or die;` — direct-access guard
`ABSPATH` is a constant WordPress defines during its own boot. If this file is requested directly (someone hits `.../fluent-community.php` in a browser) WP was never loaded, `ABSPATH` is undefined, and the script dies immediately.

- **Why it matters:** prevents leaking code / running the file outside WP. A security must-have.
- `or die` is shorthand: `defined(...)` returns false → `die` runs. You'll also see `if (!defined('ABSPATH')) exit;`.
- **MERN analogy:** like refusing to run a module unless it was `require`d by the real app, never executed standalone.

## 3. The plugin header (doc-block comment)
```php
/**
 * Plugin Name: FluentCommunity
 * ...
 */
```
This is **not a normal comment** — WordPress parses it. When WP scans the `plugins/` folder it reads these headers to build the Plugins admin list. Without a valid `Plugin Name:` line, WP does not recognize the file as a plugin.

Common header fields:

| Field | Purpose |
|-------|---------|
| `Plugin Name` | **Required.** Display name; makes WP treat file as a plugin. |
| `Description` | Shown under the name in the admin list. |
| `Version` | Used for update checks and by other code (`get_plugin_data`). |
| `Author` / `Author URI` | Credit + link. |
| `Plugin URI` | Plugin homepage. |
| `License` | Legal (WP.org requires GPL-compatible). |
| `Text Domain` | **i18n key.** Scopes translatable strings — here `fluent-community`. Every `__('text', 'fluent-community')` call references this. |
| `Domain Path` | Folder holding translation files (`.mo/.po`), here `/language`. |
| `Requires at least` / `Requires PHP` | Optional min-version gates. |

**MERN analogy:** this block is the plugin's `package.json` metadata — name, version, description — but embedded in a comment because WP reads files, not JSON.

## 4. `define(...)` — global constants
```php
define('FLUENT_COMMUNITY_PLUGIN_VERSION', '2.6.1');
define('FLUENT_COMMUNITY_PLUGIN_DIR',  plugin_dir_path(__FILE__));
define('FLUENT_COMMUNITY_PLUGIN_URL',  plugin_dir_url(__FILE__));
```
Constants are immutable globals available anywhere after this point. Plugins define a set of them up front so the rest of the codebase never hard-codes paths or versions.

- `__FILE__` = absolute path of the current file (a PHP magic constant).
- `plugin_dir_path(__FILE__)` → filesystem path to the plugin folder (for `require`).
- `plugin_dir_url(__FILE__)` → public URL to the plugin folder (for `<script>`/`<img>` src).
- `microtime(true)` → a start timestamp, used later to measure boot/render time.
- `*_DB_VERSION`, `*_MIN_PRO_VERSION` → version gates the code checks to trigger migrations or compatibility warnings.

Naming convention: `UPPER_SNAKE`, prefixed with the plugin name to avoid collisions with other plugins (no namespaces for constants — the prefix **is** the namespace).

**MERN analogy:** like a frozen `config` / `process.env` object, but resolved at boot and globally readable via the constant name.

## 5. Conditional `define` (override guard)
```php
if (!defined('FLUENTCRM_COMMUNITY_UPLOAD_DIR')) {
    define('FLUENT_COMMUNITY_UPLOAD_DIR', 'fluent-community');
}
```
Defines a constant **only if not already set**, letting site owners (or a sibling plugin) override the value in `wp-config.php` first. Redefining an existing constant throws a PHP warning, so the guard is required. Common pattern for user-tunable settings.

## 6. `require __DIR__ . '/vendor/autoload.php';` — Composer autoloader
Loads Composer's PSR-4 autoloader. After this line, any class like `FluentCommunity\App\Models\Feed` auto-resolves to `app/Models/Feed.php` on first use — no manual `require` per class.

- `__DIR__` = directory of the current file.
- `vendor/` = Composer's install dir (like `node_modules/`).
- The namespace→folder map lives in `composer.json` under `autoload.psr-4`.

**MERN analogy:** equivalent to Node's module resolution — but you must explicitly load the autoloader once; after that, `use`/class references resolve lazily.

## 7. Bootstrap kickoff
```php
call_user_func(function ($bootstrap) {
    $bootstrap(__FILE__);
}, require(__DIR__ . '/boot/app.php'));
```
Reads inside-out:

1. `require(__DIR__ . '/boot/app.php')` — this file **returns a closure** (an anonymous function). `require` yields that returned value.
2. `call_user_func(fn, $closure)` — calls the wrapper, passing the closure in as `$bootstrap`.
3. The wrapper runs `$bootstrap(__FILE__)` — invokes the real bootstrap, handing it the plugin's main-file path (needed to register activation hooks).

Why the indirection instead of just `require`? It keeps the returned closure out of the global scope and passes `__FILE__` in cleanly. The actual work — creating the `Application`, registering hooks, scheduling cron, loading modules — happens in `boot/app.php`.

**MERN analogy:** `index.js` does `const boot = require('./boot/app'); boot(__filename);` — thin entry that delegates to a bootstrap module.

→ Continue in [BOOTSTRAP.md](BOOTSTRAP.md) for what `boot/app.php` does.

---

## Key takeaways
- The entry file is **discovery + setup only** — header for WP, constants for the codebase, autoloader for classes, then hand off to bootstrap.
- The header comment is machine-read metadata, not decoration.
- `defined('ABSPATH') or die;` at the top of nearly every PHP file blocks direct access.
- Constants are prefixed to fake a namespace and avoid plugin collisions.
- Real logic never lives here — it starts in `boot/app.php`.
