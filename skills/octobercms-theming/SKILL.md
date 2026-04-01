---
name: octobercms-migration-theming
description: Converts scraped HTML into OctoberCMS v4 theme files. Auto-trigger when working with files in a .migration/ directory, when generating OctoberCMS theme layouts/partials/pages from HTML, or when the user mentions site migration, HTML conversion, theme generation, or wp-migrate.
---

# HTML to OctoberCMS Theme Conversion

This skill handles converting raw HTML (scraped from a live website) into production-ready OctoberCMS v4 theme files.

**Prerequisite:** The `softappstudio/octobercms` plugin must be installed for full OctoberCMS documentation access. Use it to look up any OctoberCMS syntax you're unsure about.

## Theme File Format

Every OctoberCMS template file (layout, page, partial) has up to 3 sections separated by `==`:

```
[configuration section - INI format]
==
<?php // PHP code section (optional) ?>
==
{# Twig markup section #}
```

## Layout Conversion Rules

When converting a website's outer HTML shell into a layout:

**File:** `themes/<name>/layouts/default.htm`

```
description = "Default Layout"

[sitePicker]
[localePicker]
==
<!DOCTYPE html>
<html lang="{{ this.site.locale }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ this.page.meta_title ? this.page.meta_title|trans : this.page.title|trans }} - Site Name</title>
    <meta name="description" content="{{ this.page.meta_description }}">
    <meta property="og:title" content="{{ this.page.meta_title ? this.page.meta_title|trans : this.page.title|trans }} - Site Name">
    <meta property="og:description" content="{{ this.page.meta_description }}">
    {% for site in sitePicker.sites %}
        <link rel="alternate" hreflang="{{ site.locale }}" href="{{ site.url }}" />
    {% endfor %}
    {% styles %}
    <link rel="stylesheet" href="{{ 'assets/css/style.css'|theme }}">
</head>
<body class="{{ this.page.cssClass }}">
    {% partial 'header' %}

    {% page %}

    {% partial 'footer' %}

    {% scripts %}
    <script src="{{ 'assets/js/main.js'|theme }}"></script>
</body>
</html>
```

**Rules:**
- `{% styles %}` goes in `<head>` BEFORE custom stylesheets
- `{% scripts %}` goes before `</body>` BEFORE custom scripts
- `{% page %}` marks where page content renders — exactly ONE per layout
- `{% partial 'name' %}` for each reusable block
- Use `{{ this.site.locale }}` for the `<html lang>` attribute — NOT a hardcoded language
- **Title translation:** Use `{{ this.page.meta_title ? this.page.meta_title|trans : this.page.title|trans }}` — NOT `{{ this.page.meta_title ?: this.page.title }}`. The `|trans` filter is essential for multilingual sites so titles are translated in the layout.
- Apply `|trans` to og:title as well
- Add `[sitePicker]` and `[localePicker]` components for multilingual sites
- Add `<link rel="alternate" hreflang>` tags for SEO
- Keep external CDN links (Google Fonts, Font Awesome, jQuery CDN) as-is
- Convert local asset paths to `{{ 'assets/path/file.ext'|theme }}`

## Partial Conversion Rules

When extracting a reusable HTML block into a partial:

**File:** `themes/<name>/partials/<name>.htm`

```
{# No configuration section needed for simple partials #}
<header class="site-header">
    <a href="/" class="logo">
        <img src="{{ 'assets/images/logo.png'|theme }}" alt="Site Name">
    </a>
    <nav class="main-nav">
        <ul>
            <li><a href="/">Home</a></li>
            <li><a href="/about">About</a></li>
        </ul>
    </nav>
</header>
```

**Rules:**
- Partials have NO configuration section by default (just the Twig markup)
- Partials CAN accept variables: `{% partial 'hero' title='Welcome' image=heroImage %}`
- If a partial needs variables, declare them with `{% props %}`:
  ```
  {% props title, subtitle = 'Default subtitle' %}
  <section class="hero">
      <h1>{{ title }}</h1>
      <p>{{ subtitle }}</p>
  </section>
  ```
- Navigation partials: hardcode links initially, add component wiring later
- Keep all CSS classes that are referenced by the stylesheets

## Page Conversion Rules

When converting unique page content:

**File:** `themes/<name>/pages/<name>.htm`

```
url = "/a-propos/"
layout = "default"
title = "À propos"

[viewBag]
localeUrl[en] = "/about-us/"
localeUrl[fr] = "/a-propos/"
meta_title = "À propos - Site Name"
meta_description = "En savoir plus sur notre entreprise"
==
<section class="page-about">
    <h1>{{ 'À propos'|trans }}</h1>
    <div class="content">
        <p>Page content here...</p>
    </div>
</section>
```

**URL patterns for dynamic pages:**
- Static: `url = "/about"`
- With parameter: `url = "/blog/:slug"`
- Optional parameter: `url = "/blog/:slug?"`
- Wildcard: `url = "/blog/:category*/:slug"`

**Multilingual URL translation with `localeUrl` (requires `rainlab/translate-plugin`):**
```
[viewBag]
localeUrl[en] = "/project/:slug"
localeUrl[fr] = "/realisation/:slug"
```
The `localeUrl` in viewBag tells RainLab.Translate what the URL should be for each locale. The base `url` stays in the primary language. The route prefix (e.g., `/en`) is added automatically by OctoberCMS multisite — do NOT include it in `localeUrl`.

**Rules:**
- Every page MUST have `url` and `layout`
- `title` is used as fallback for `<title>` tag
- Only include the content area — NOT the layout/header/footer
- For listing pages, wrap content in loop structures (to be wired to components later)
- For detail pages, use Twig variables (to be populated by components later)
- **For multilingual sites:** add `localeUrl[xx]` entries in `[viewBag]` for EVERY page — including static pages and dynamic `:slug` pages
- **Use `|trans` filter** on user-visible text in templates (headings, button labels, etc.)
- **Add title translations** to `lang/en.json` (or other locale files) including `meta_title` values like `"À propos - My Company": "About us - My Company"`

## Asset Path Conversion

Convert ALL local asset references:

| Original | Converted |
|----------|-----------|
| `/wp-content/themes/xyz/style.css` | `{{ 'assets/css/style.css'\|theme }}` |
| `/wp-content/uploads/2024/01/photo.jpg` | `{{ 'assets/images/photo.jpg'\|theme }}` |
| `../images/logo.png` | `{{ 'assets/images/logo.png'\|theme }}` |
| `https://fonts.googleapis.com/...` | Keep as-is (external CDN) |
| `https://cdn.jsdelivr.net/...` | Keep as-is (external CDN) |
| `data:image/svg+xml,...` | Keep as-is (inline) |

**In CSS files:** Keep relative paths within CSS since they resolve relative to the CSS file location inside `assets/css/`. **CRITICAL: Also check for `@font-face` declarations** that reference the original domain (e.g., `url(https://oldsite.com/.../font.woff2)`) — download these fonts to `assets/fonts/` and rewrite to `url(../fonts/font.woff2)`. Original domain fonts will fail due to CORS.

**File attachment images in Twig:** When displaying images from `$attachOne`/`$attachMany` relations (not static theme assets), use `.url`:
```twig
{# Correct — for model file attachments #}
<img src="{{ model.featured_image.url }}" alt="">

{# Wrong — .path does not exist on File model #}
<img src="{{ model.featured_image.path }}" alt="">

{# For static theme assets, continue using |theme filter #}
<img src="{{ 'assets/images/logo.svg'|theme }}" alt="">
```

## WordPress Cleanup Checklist

When converting WordPress HTML, remove or replace:

**Remove entirely:**
- `<!-- wp:paragraph -->` and all Gutenberg block comments
- `<link rel="dns-prefetch" href="//s.w.org">`
- `wp-emoji-release.min.js` and emoji detection scripts
- `wp-embed.min.js`
- `<meta name="generator" content="WordPress...">`
- `?ver=X.X.X` query strings on asset URLs
- `wp-json` link tags
- `xmlrpc` link tags
- `wlwmanifest` link tags
- REST API discovery link tags
- WordPress admin bar markup and CSS
- `nonce` fields and WordPress CSRF tokens
- WooCommerce cart fragments script (unless migrating e-commerce)

**Replace:**
- `wp-content/themes/*/` paths → `assets/` with `|theme` filter
- `wp-content/uploads/` paths → `assets/images/` with `|theme` filter
- `wp-includes/` script/style paths → remove or replace with CDN equivalents
- WordPress body classes → keep only semantic ones, remove `wp-*`, `logged-in`, `admin-bar`

**Keep:**
- All CSS class names referenced by stylesheets (even if they look WordPress-y like `entry-content`)
- Third-party scripts (analytics, chat widgets, etc.)
- Font loading (Google Fonts, Adobe Fonts, etc.)
- Any JavaScript that powers interactive elements (sliders, accordions, mobile menus)

## Page Builder Cleanup

WordPress page builders (Elementor, WPBakery, Divi, Kadence) produce deeply nested HTML. When converting:

1. **Identify the grid system** — most builders use a section > container > column > widget pattern
2. **Simplify nesting** where possible — remove wrapper divs that only exist for the builder's JS
3. **Keep structural classes** that are referenced by the CSS — removing them breaks the layout
4. **Remove builder-specific data attributes** (`data-elementor-*`, `data-vc-*`, etc.)
5. **Remove inline styles** that duplicate what the CSS already does — but keep inline styles that are unique (e.g., background images set per-section)

## Tailor Blueprint Patterns

When generating Tailor blueprints for content groups:

**Stream** (blog, news, portfolio — many time-ordered entries):
```yaml
uuid: <generate-with-php-or-uuidgen>
handle: Content\BlogPost
type: stream
name: Blog Post
drafts: true

fields:
    title:
        label: Title
        type: text
    slug:
        label: Slug
        type: text
        preset:
            field: title
            type: slug
    content:
        label: Content
        type: richeditor
    featured_image:
        label: Featured Image
        type: mediafinder
        mode: image
    published_at:
        label: Published Date
        type: datepicker
```

**Single** (homepage, about — one editable record):
```yaml
uuid: <generate>
handle: Content\Homepage
type: single
name: Homepage

fields:
    hero_title:
        label: Hero Title
        type: text
    hero_subtitle:
        label: Hero Subtitle
        type: textarea
    hero_image:
        label: Hero Image
        type: mediafinder
        mode: image
```

**Global** (site settings, social links):
```yaml
uuid: <generate>
handle: Site\Config
type: global
name: Site Configuration

fields:
    site_name:
        label: Site Name
        type: text
    phone:
        label: Phone
        type: text
    email:
        label: Email
        type: text
    social_facebook:
        label: Facebook URL
        type: text
    social_instagram:
        label: Instagram URL
        type: text
```

## Multilingual Sites — Critical Rules

When the site has multiple languages, these rules are essential. Failure to follow them causes broken URLs, empty detail pages, and untranslated titles.

### 1. Install RainLab.Translate

```bash
composer require rainlab/translate-plugin
php artisan october:migrate
```

This plugin provides `localeUrl` viewBag support and the `localePicker` component. Without it, URL translation does not work.

### 2. Layout Must Include Both Components

```
[sitePicker]
[localePicker]
```

`sitePicker` provides `sitePicker.sites` for language switcher links and hreflang tags. `localePicker` enables the `localeUrl` routing.

### 3. Never Hardcode Internal Links

**Wrong:**
```twig
<a href="/realisation/{{ realisation.slug }}/">{{ realisation.title }}</a>
```

**Correct:**
```twig
<a href="{{ 'realisation'|page({slug: realisation.slug}) }}">{{ realisation.title }}</a>
```

The `|page` filter generates locale-aware URLs that respect `localeUrl`. Hardcoded links will always point to the primary language URL and ignore the `/en/` prefix.

### 4. Component Slug Lookup Must Handle Translations

OctoberCMS Translatable trait stores primary language values in the model's base column and translations in a separate table (`system_translate_attributes`). A simple `where('slug', $slug)` will NOT find translated slugs.

Components must search both:

```php
public function realisation()
{
    $slug = $this->property('slug');

    // Primary language slug is in the base column; translated slugs are in the translations table
    return Realisation::where('slug', $slug)->published()->first()
        ?: Realisation::whereTranslation('slug', 'en', $slug)->published()->first();
}
```

The `whereTranslation` signature is: `whereTranslation($attribute, $locale, $value)`.

### 5. Seeding Translated Content — Strip HTML from Plain Text Fields

When seeding English translations for plain-text fields (like `excerpt`), strip HTML tags:

```php
$model->setTranslation('excerpt', 'en', strip_tags($englishExcerpt));
$model->save();
```

WYSIWYG fields (`content`) can keep HTML. But plain text fields (`excerpt`, `title`, `slug`) must NOT contain HTML tags — they will render as raw markup in the backend and frontend.

### 6. Theme Language Files

Create `themes/<name>/lang/en.json` with translations for ALL user-visible strings:

```json
{
    "À propos": "About us",
    "Réalisations": "Projects",
    "À propos - My Company": "About us - My Company",
    "Contactez-nous": "Contact us"
}
```

This file uses the primary language string as the key and the translation as the value. The `|trans` Twig filter looks up these keys.

## Wiring Components to Pages

**Tailor collection listing:**
```
url = "/blog"
layout = "default"
title = "Blog"

[collection]
handle = "Content\BlogPost"
recordsPerPage = 10
pageNumber = "{{ :page }}"
sortColumn = "published_at"
sortDirection = "desc"
==
<div class="blog-listing">
    {% for post in collection.records %}
        <article>
            <h2><a href="{{ 'blog-post'|page({slug: post.slug}) }}">{{ post.title }}</a></h2>
            <p>{{ post.content|striptags|truncate(150) }}</p>
        </article>
    {% endfor %}

    {% if collection.records.lastPage > 1 %}
        {{ pager(collection.records) }}
    {% endif %}
</div>
```

**Tailor section detail:**
```
url = "/blog/:slug"
layout = "default"
title = "Blog Post"

[section]
handle = "Content\BlogPost"
identifier = "slug"
value = "{{ :slug }}"
==
{% set post = section.record %}
<article class="blog-post">
    <h1>{{ post.title }}</h1>
    <time>{{ post.published_at|date('F j, Y') }}</time>
    {% if post.featured_image %}
        <img src="{{ post.featured_image|media }}" alt="{{ post.title }}">
    {% endif %}
    <div class="content">
        {{ post.content|raw }}
    </div>
</article>
```

**Tailor global access (in layout or partial):**
```
[global siteConfig]
handle = "Site\Config"
==
<footer>
    <p>{{ siteConfig.record.phone }}</p>
    <a href="{{ siteConfig.record.social_facebook }}">Facebook</a>
</footer>
```
