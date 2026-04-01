---
description: Migrate any website into an OctoberCMS v4 theme. Provide a URL to scrape, analyze, and generate a complete theme.
---

# Site-to-October Migration

You are migrating the website at `$ARGUMENTS` into an OctoberCMS v4 theme.

Your job is to execute a multi-phase pipeline that takes a live website URL, scrapes it, analyzes its structure, and generates a complete OctoberCMS v4 theme with layouts, partials, pages, and assets. You must produce production-ready Twig templates that faithfully replicate the original site's appearance on both desktop and mobile.

## Phase 0: Environment Setup

Before doing any scraping or analysis, you MUST verify and prepare the environment. Run these checks in order and STOP at any failure until resolved.

### Step 0A: Check Tools

Verify `curl` and `wget` are available (run `which curl wget`). These are the scraping tools used by the plugin. They should be pre-installed on most systems. If either is missing, ask the user to install them before continuing.

### Step 0B: Check for OctoberCMS Project

Check if the current directory is an OctoberCMS v4 project by looking for `artisan` and `composer.json` containing `october/rain`.

**If NOT an OctoberCMS project**, tell the user:

```
You're not inside an OctoberCMS project. I need one to generate the theme into.

Want me to create a fresh OctoberCMS v4 project here? I'll run:
  composer create-project october/october .
  php artisan october:install

Or if you have an existing project elsewhere, navigate there first.
```

**STOP and wait for the user's answer.** If they say yes, run the composer create-project and artisan install commands, then continue.

### Step 0C: Theme Name

Derive a theme name from the URL (e.g., `solmatech` from `solmatech.ca`, `bellevie` from `belleviespa.com`). Confirm with the user:

```
I'll create the theme as "solmatech". Sound good, or do you want a different name?
```

**STOP and wait for confirmation** before proceeding to Phase 1.

## Phase 1: Discovery

**Goal:** Map the entire site structure and identify all unique page templates.

### Step 1A: URL Discovery

1. Fetch `/sitemap.xml` via curl — parse all `<loc>` URLs
2. If no sitemap, fetch the homepage and extract all internal links recursively (limit to 100 pages)
3. Save discovered URLs to `.migration/urls.txt`

### Step 1B: URL Grouping

Analyze the discovered URLs and group them by pattern:
- Static pages: `/about`, `/contact`, `/privacy` (unique URLs, no pattern)
- Collection pages: `/blog/*`, `/services/*`, `/team/*` (multiple URLs sharing a prefix)
- Listing pages: `/blog`, `/services` (index/archive pages)
- The homepage: `/`

Save the grouping to `.migration/url-groups.json` with this structure:
```json
{
  "groups": [
    {
      "name": "homepage",
      "pattern": "/",
      "type": "single",
      "urls": ["/"],
      "sample_url": "/"
    },
    {
      "name": "blog",
      "pattern": "/blog/*",
      "type": "collection",
      "urls": ["/blog/post-1", "/blog/post-2"],
      "listing_url": "/blog",
      "sample_url": "/blog/post-1"
    }
  ]
}
```

### Step 1C: WordPress API Probe (Optional Bonus)

Try fetching `<URL>/wp-json/wp/v2/` — if it returns a valid response:
- Fetch posts, pages, categories, custom post types
- Save structured content to `.migration/wp-api/`
- This is a bonus — do NOT depend on it. The HTML scraping is the primary source.

## Phase 2: Scraping

**Goal:** Download HTML and assets for all unique page templates.

### Step 2A: Scrape Key Pages

For each URL group, scrape the **sample URL** (one per group) plus any listing URLs for template analysis. **IMPORTANT: For collection groups (blog/*, services/*, etc.), also scrape ALL individual pages** so you can extract real content for seeding the database later. Scraping only one sample per group will leave most records empty.

```bash
mkdir -p .migration/pages/<group-name>
curl -sL -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36" "<FULL_URL>" -o .migration/pages/<group-name>/index.html
```

Also scrape the same pages with a mobile user-agent to detect responsive differences:
```bash
curl -sL -A "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1" "<FULL_URL>" -o .migration/pages/<group-name>/mobile.html
```

### Step 2B: Download Assets

Download all CSS, JS, images, and fonts referenced in the scraped HTML. **IMPORTANT: Also check inside CSS files for `@font-face` declarations pointing to the original domain** — download those fonts locally and rewrite the CSS `url()` paths to use relative paths (e.g., `../fonts/`). Fonts served from the original domain will be blocked by CORS on the new site.

**Approach:** Parse the downloaded HTML files, extract all asset URLs (stylesheets, scripts, images, fonts), then download each with curl. Organize by type:

```bash
mkdir -p .migration/assets/{css,js,images,fonts}
```

For bulk download, wget can mirror assets efficiently:
```bash
wget -q -P .migration/assets/ -nd -r -l 1 -A css,js,png,jpg,jpeg,gif,svg,webp,woff,woff2,ttf,eot,ico <URL>
```

Or extract asset URLs from the HTML and download them individually with curl.

Organize downloaded assets into:
```
.migration/assets/
  css/
  js/
  images/
  fonts/
```

## Phase 3: Analysis

**Goal:** Decompose the HTML into reusable OctoberCMS theme components.

Read ALL the scraped HTML files. Compare them to identify:

### Step 3A: Layout Detection

Find the **common outer structure** shared across all pages:
- `<head>` content (meta tags, CSS links, fonts)
- Header area (logo, navigation, top bar)
- Footer area (footer links, copyright, scripts)
- Any wrapper/container structure

This becomes the OctoberCMS **layout**.

### Step 3B: Partial Identification

Find **reusable blocks** that appear on multiple pages:
- **Header** — logo, navigation menu, mobile menu toggle, search, CTA buttons
- **Footer** — footer columns, links, social icons, copyright
- **Navigation** — main menu, mobile menu/hamburger, dropdown submenus
- **Sidebar** — if present on some pages
- **Any other repeated blocks** — newsletter signup, CTA banner, etc.

Each becomes an OctoberCMS **partial**.

### Step 3C: Page Content Isolation

For each page template, identify the **unique content area** — the part that changes between pages. This is what goes into OctoberCMS **page** files.

### Step 3D: Content Extraction

For collection groups (blog/*, services/*, etc.), extract the structured content from each page:
- Title, heading
- Body/description
- Featured image
- Date, author
- Categories, tags
- Any other fields

Save to `.migration/content/<group-name>/` as JSON files.

### Step 3E: Migration Report

Generate `.migration/report.md` summarizing:
- Total pages found
- URL groups identified
- Partials identified
- Content types detected with field structures
- Suggested OctoberCMS approach per group:
  - **Static pages** → inline in theme pages
  - **Blog-like content** → RainLab.Blog plugin or Tailor stream blueprint
  - **Repeating content** (services, team, portfolio) → Tailor stream blueprint
  - **Single configurable pages** (home, about) → Tailor single blueprint or static page
  - **Global elements** (site settings, social links) → Tailor global blueprint

## Phase 4: Theme Generation

**Goal:** Create a complete, working OctoberCMS v4 theme.

### Step 4A: Theme Scaffold

Create the theme directory:
```
themes/<theme-name>/
  theme.yaml
  layouts/
  partials/
  pages/
  content/
  assets/
    css/
    js/
    images/
    fonts/
```

Create `theme.yaml`:
```yaml
name: '<Theme Name>'
description: 'Migrated from <original URL>'
author: 'SoftAppStudio'
homepage: '<original URL>'
```

### Step 4B: Asset Migration

Copy all downloaded assets into `themes/<theme-name>/assets/`. Rewrite all asset paths in the HTML to use the OctoberCMS `|theme` filter:

- `href="/wp-content/themes/xyz/style.css"` → `href="{{ 'assets/css/style.css'|theme }}"`
- `src="/images/logo.png"` → `src="{{ 'assets/images/logo.png'|theme }}"`
- External CDN resources (Google Fonts, Font Awesome, jQuery, etc.) → keep as-is

### Step 4C: Layout Generation

Create `layouts/default.htm` from the common structure identified in Phase 3A.

The layout MUST use these OctoberCMS conventions:
```twig
description = "Default Layout"
==
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ this.page.meta_title ?: this.page.title }} - Site Name</title>
    <meta name="description" content="{{ this.page.meta_description }}">
    {% styles %}
    <link rel="stylesheet" href="{{ 'assets/css/style.css'|theme }}">
</head>
<body>
    {% partial 'header' %}

    {% page %}

    {% partial 'footer' %}

    {% scripts %}
    <script src="{{ 'assets/js/main.js'|theme }}"></script>
</body>
</html>
```

If multiple distinct layouts exist (e.g., pages with sidebar vs. full-width), create separate layout files.

### Step 4D: Partial Generation

Create a partial `.htm` file for each reusable block identified in Phase 3B.

For the navigation partial, extract the menu structure and create proper HTML. Use hardcoded links initially — these can be wired to a CMS menu component later.

For each partial:
- Clean up WordPress-specific classes (wp-*, elementor-*, etc.)
- Keep the CSS class names that are referenced by the downloaded stylesheets
- Rewrite asset paths with `|theme` filter
- Remove WordPress-specific JavaScript hooks (wp-embed, wp-emoji, etc.)

### Step 4E: Page Generation

Create a page `.htm` file for each unique template:

```twig
url = "/about"
layout = "default"
title = "About Us"
description = "About page"

[viewBag]
meta_title = "About Us"
meta_description = "Learn about our company"
==
<div class="page-content">
    <!-- Converted page content here -->
</div>
```

For **collection detail pages** (blog post, service detail), use URL parameters:
```twig
url = "/blog/:slug"
```

For **listing pages**, include pagination structure.

### Step 4F: WordPress Cleanup

Remove or replace all WordPress-specific elements:
- Remove `wp-content`, `wp-includes` references
- Remove WordPress JavaScript (`wp-embed.min.js`, `wp-emoji-release.min.js`, etc.)
- Remove WordPress comments markup (`<!-- wp:... -->`)
- Remove Gutenberg/block editor wrapper classes unless they're styled
- Remove WooCommerce cart/account widget markup (unless e-commerce is being migrated)
- Replace WordPress-generated image `srcset` with simple `src`
- Remove `?ver=` query strings from asset URLs
- Clean up excessive `<div>` nesting from page builders (Elementor, WPBakery, Divi)

### Step 4G: Mobile/Responsive Verification

Compare desktop and mobile HTML from Phase 2A:
- If the site is responsive (same HTML, CSS media queries): the theme should work as-is
- If the site serves different HTML for mobile: note this in the report and ensure the CSS media queries are included
- Ensure the mobile hamburger menu / navigation toggle has proper JS included

## Phase 5: Content Strategy

**Goal:** Let the user decide how to manage each content group, then generate the appropriate structures.

### Step 5A: Present Options

Show the user the migration report from Phase 3E. For each content group, present:
1. The group name and URL pattern
2. Number of entries
3. Detected fields (title, body, image, date, etc.)
4. Recommended approach (RainLab.Blog, Tailor blueprint, static page)

Also present the option of a **full plugin with models/controllers** instead of Tailor:
- Better for production sites — custom list views, filters, scopes, relations
- Full control over backend UI and business logic
- Standard OctoberCMS pattern any developer can maintain
- More work upfront but better long-term

Ask the user: **"How do you want to handle each group? Tailor blueprints (quick), full plugin (production-grade), or static pages?"**

### Step 5B: Generate Content Structures

Based on user choices:

**For Tailor stream blueprints** (blog, news, portfolio):
Create `themes/<theme-name>/blueprints/<group>.yaml`:
```yaml
uuid: <generate-uuid>
handle: Content\<GroupName>
type: stream
name: <Group Name>
drafts: true

fields:
    title:
        label: Title
        type: text
    # ... fields detected from content extraction
```

**For Tailor single blueprints** (homepage, about):
```yaml
uuid: <generate-uuid>
handle: Content\<PageName>
type: single
name: <Page Name>

fields:
    # ... editable fields from the page
```

**For Tailor global blueprints** (site settings, social links):
```yaml
uuid: <generate-uuid>
handle: Site\Config
type: global
name: Site Configuration

fields:
    # ... global settings
```

**For a full plugin with models/controllers:**
Create a single plugin (e.g., `Author.Content`) with models, controllers, and components for each content type. Follow these critical rules:

1. Use `php artisan create:plugin`, `create:model`, `create:controller` to scaffold
2. **version.yaml format** — MUST use proper YAML list syntax:
   ```yaml
   v1.0.1:
       - 'First version'
       - create_table_one.php
       - create_table_two.php
   ```
   NOT `v1.0.1: Description` on one line — that makes October treat migrations as a comment and skip them.

3. **NEVER create a database column with the same name as an `$attachOne`/`$attachMany` relation.** This causes a naming collision where the column value shadows the relation, making file attachments invisible. If you need `featured_image` as an attachment, do NOT add a `featured_image` string column to the migration.

4. **File attachments — correct way to attach files programmatically:**
   ```php
   $file = new \System\Models\File;
   $file->fromFile('/path/to/image.jpg');
   $file->is_public = true;
   $file->save();
   $model->featured_image()->add($file);
   ```
   Do NOT use `$model->featured_image = (new File)->fromFile(...)` in seeders — it won't persist.

5. **In Twig templates, use `.url` to get the image URL, not `.path`:**
   ```twig
   <img src="{{ model.featured_image.url }}" alt="">
   ```

6. **Add `protected $guarded = [];`** to all models to allow mass assignment in seeders.

7. **Register components** in Plugin.php for frontend pages (List + Detail per content type).

8. **Create a seeder migration** (v1.0.2) that populates all records with real scraped content and attaches images.

**For RainLab.Blog:**
- Ensure the plugin is installed: `composer require rainlab/blog-plugin`
- Wire the `blogPosts` and `blogPost` components into the listing and detail pages

### Step 5C: Wire Components to Pages

Update the page files to use the appropriate components:

For Tailor collections:
```twig
url = "/blog"
layout = "default"
title = "Blog"

[collection]
handle = "Content\BlogPost"
recordsPerPage = 10
==
{% set posts = collection.records %}
<!-- listing markup -->
```

For Tailor sections (detail pages):
```twig
url = "/blog/:slug"
layout = "default"
title = "Blog Post"

[section]
handle = "Content\BlogPost"
identifier = "slug"
==
{% set post = section.record %}
<!-- detail markup -->
```

### Step 5D: Import Content

If content was extracted in Phase 3D and Tailor blueprints were generated:
1. Run `php artisan tailor:migrate` to create tables
2. Create a temporary artisan command or seeder to import the JSON content
3. Or instruct the user to manually enter content via the backend

## Phase 6: Finalization

1. Run `php artisan theme:use <theme-name>` to activate the theme
2. Run `php artisan october:migrate` to ensure all migrations are up to date
3. Tell the user to run `php artisan serve` and check the result
4. Ask what they want to adjust

## Important Rules

- **Pixel-perfect first, optimize later.** Match the original site's appearance exactly before cleaning up or improving anything.
- **Real content, not placeholders.** Use actual text, images, and data from the scraped site.
- **Mobile-first CSS.** Ensure all responsive breakpoints from the original CSS are preserved.
- **Clean Twig syntax.** Follow OctoberCMS v4 conventions — use `{% partial %}`, `{% page %}`, `{% placeholder %}`, `|theme` filter, etc.
- **No unnecessary plugins.** Don't install plugins unless the user explicitly requests them for a content group.
- **Preserve functionality.** If the site has forms, sliders, accordions, tabs — keep the JavaScript that powers them.
- **Work incrementally.** Complete each phase before moving to the next. Show progress to the user at each phase boundary.
- **Ask before deciding on content strategy.** Phase 5 is interactive — never auto-decide whether to use Tailor vs plugin vs RainLab vs static.
- **Download ALL assets including fonts.** Check CSS `@font-face` rules for font URLs pointing to the original domain — these WILL break due to CORS. Download fonts locally and rewrite CSS paths.
- **Scrape ALL collection pages for content.** Don't just scrape one sample per template — scrape every page in the collection so you can seed the database with real content.
- **Check APP_URL in .env.** After migration, update `APP_URL` to match the local dev domain (e.g., `http://sitename.test`). File attachment URLs are generated from this value.
- **Storage permissions.** After setup, ensure `storage/` is writable by the web server: `chown -R www-data:www-data storage && chmod -R 775 storage`.
