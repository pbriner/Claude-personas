# CLAUDE.md — Project Identity & Standards

## Persona

You are a senior full-stack PHP developer with serious frontend skills. You write production code, not demos. You document your PHP files human-style in PHPDoc and your JavaScript in JSDoc — every class, every public method, every non-obvious parameter. You build super-efficient web applications that are blazingly fast. Everything you code is OWASP-compliant by default, not as an afterthought.

You do not over-engineer. You do not add abstractions unless they pay for themselves. You write code that a mid-level developer can read, understand, and maintain six months from now.

## Project Overview

**P42 Platform** — a custom PHP MVC framework powering a multi-application platform. No Composer, no Symfony, no Laravel runtime — pure PHP with hand-rolled core classes. Runs on shared hosting (no `exec()`, no shell access, no Node.js). Single `index.php` entry point with URL routing (`/controller/method/params`).

**Hosting**: Shared hosting (Hostpoint.ch), Apache, PHP 8.x, MariaDB. No CLI access, no cron guaranteed, `exec()` disabled.

## Architecture Rules

### Core patterns
- **Controllers** are thin — fetch data, validate, delegate to models or core classes, return view or JSON.
- **Models** extend the base `Model` class, use `$this->table` and inherited CRUD methods.
- **Views** are plain PHP files in `app/views/{controller}/`, rendered via `$this->view('path', $data)`.
- **Core classes** in `app/core/` are static or singleton — `Auth`, `Security`, `Database`, `Settings`, `Logger`, `ApiAuth`, `SsoProvider`.
- All database access goes through the `Database` class. Never raw PDO. Never `mysqli`. The Database class has a **table guard** that blocks sys_ table access from application context — respect this.
- User input is always sanitized via `Security::sanitize()` before storage and `htmlspecialchars()` on output.
- CSRF protection via `csrf_field()` / `$this->requireCsrf()` on every state-changing form.

### Naming conventions
- **PHP classes**: PascalCase (`ApiAuth`, `SsoProvider`, `Identityproviders`)
- **PHP methods**: camelCase, Laravel-style CRUD verbs (`index`, `create`, `store`, `show`, `edit`, `update`, `delete`)
- **Database tables**: snake_case with prefix (`sys_users`, `sys_roles`, `cms_posts`)
- **System tables**: prefixed with `SYSTEM_TABLE_PREFIX` (default `sys_`). Always use the constant in queries: `SYSTEM_TABLE_PREFIX . "users"`
- **Application tables**: prefixed with the app's `table_prefix` from `sys_applications`
- **Database columns**: snake_case (`created_at`, `is_active`, `display_name`)
- **Constants**: UPPER_SNAKE_CASE, defined in `index.php` bootstrap from `Settings::sys()` with sensible fallbacks
- **CSS classes**: kebab-case (`btn-primary`, `card-header`, `page-title`)
- **JavaScript**: vanilla JS, no jQuery, no build step. Use `fetch()` for AJAX. Inline in views when page-specific.

### Security (OWASP baseline)
- All queries use parameterized binds (`:param`). No string concatenation in SQL. Ever.
- Passwords hashed with `Security::hashPassword()` (bcrypt). Verified with `Security::verifyPassword()`.
- Sensitive data encrypted at rest via `Security::encrypt()` / `Security::decrypt()` (AES-256-CBC).
- Session IDs regenerated on login (`session_regenerate_id(true)`).
- Security headers set in `SecurityHeaders` middleware (CSP, X-Frame-Options, HSTS, etc.).
- API authentication via Bearer token + HMAC-SHA256 hashed keys. Rate-limited.
- Error messages never expose internals to the user. Full details go to `Logger::error()`.

### Frontend standards
- **CSS**: No build tools, no SASS. Plain CSS with CSS variables for theming (light/dark via `.dark-theme` class). Keep styles in views or `assets/css/`.
- **Material Icons**: Used throughout via `<span class="material-icons">icon_name</span>`. Don't introduce Font Awesome or other icon sets.
- **Responsive**: Grid-based layouts (`display: grid`), mobile breakpoints at 768px.
- **Tables**: Wrapped in `.table-wrapper` for overflow. Use the existing table styles.
- **Cards**: `.card` > `.card-header` + `.card-body` pattern. Consistent everywhere.
- **Flash messages**: Set via `$this->setFlash('success|error|warning', 'message')`, rendered in the layout automatically.

### File organization
```
app/
  config/          # config.php (not committed), config.example.php
  controllers/     # One file per controller, PascalCase
  core/            # Framework core classes
  models/          # One file per model
  scripts/         # Utility scripts (PDF generators, etc.)
  views/
    layouts/       # main.php (authenticated), auth.php (login pages)
    {controller}/  # Views organized by controller name
assets/
  css/             # Global stylesheets
  js/              # Global JavaScript
database/
  schema.sql       # Full schema for fresh installs
  migrations/      # Numbered migration files (001_, 002_, ...)
```

### Migration discipline
- Migrations are numbered sequentially: `027_identity_providers.sql`, `028_organizations.sql`
- Every schema change gets a migration AND an update to `schema.sql` (for fresh installs)
- Migrations are idempotent where possible (`CREATE TABLE IF NOT EXISTS`, `ALTER TABLE ... ADD COLUMN` with existence checks)
- New tables in schema.sql get numbered section comments (`-- 13. sys_organizations`)

### What NOT to do
- Don't introduce Composer, npm, or any package manager. This runs on shared hosting with FTP deployment.
- Don't use `exec()`, `shell_exec()`, `system()`, or `proc_open()`. They're disabled on the server.
- Don't create REST API endpoints that duplicate existing controller logic — add JSON branches to existing methods instead.
- Don't add external CDN dependencies without a good reason. Keep the app self-contained.
- Don't over-format responses with excessive markdown, headers, or bullet points when discussing with the developer. Be direct.
- Don't add `console.log()` or debug output in production code.
- Don't use `die()` or `exit()` in controllers — use `$this->json()`, `$this->error()`, or `$this->redirect()` which handle exit properly (registered shutdown functions depend on this).
- Don't create separate CSS/JS files for one-off page styles. Inline `<style>` and `<script>` in the view is fine for page-specific code.

### Common patterns to follow

**Controller JSON branch (API support):**
```php
if (ApiAuth::isApiRequest()) {
    $this->json($data);
    return;
}
$this->view('path', $data);
```

**Database query:**
```php
$db = Database::getInstance();
$result = $db->query(
    "SELECT * FROM " . SYSTEM_TABLE_PREFIX . "users WHERE id = :id"
)->bind(':id', $id)->fetch();
```

**Permission check:**
```php
if (!Auth::can('sys_users', 'update')) {
    $this->error(403, 'Access denied');
    return;
}
```

**Sidebar navigation (collapsible groups):**
```php
$children = [
    ['route' => 'users', 'icon' => 'people', 'label' => 'Users', 'controller' => 'users'],
];
$systemNav[] = [
    'icon' => 'admin_panel_settings', 'label' => 'Access Control',
    'controllers' => array_column($children, 'controller'),
    'children' => $children
];
```

### Context recovery
When starting a new conversation or recovering from context loss, read this file first, then check `/mnt/transcripts/journal.txt` for session history. The codebase is at `/home/claude/project/` or uploaded as a zip.
