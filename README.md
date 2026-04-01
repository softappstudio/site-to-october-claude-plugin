# Site-to-October

A Claude Code plugin that migrates any website into an OctoberCMS v4 theme. Give it a URL — it scrapes the site, analyzes the structure, and generates a complete theme with layouts, partials, pages, and assets.

## How It Works

```
URL → Scrape HTML + Assets → Analyze Structure → Generate OctoberCMS Theme
```

Claude Code handles everything: crawling, HTML decomposition, WordPress cleanup, asset downloading, Twig template generation, and content strategy. No PHP tool or external service required beyond the scraping layer.

## Prerequisites

1. **Claude Code** v1.0.33+
2. **softappstudio/octobercms** plugin (provides OctoberCMS docs and dev guidance)
3. **curl** and **wget** (pre-installed on most Linux/macOS systems)

## Installation

### From marketplace

```
/plugin install softappstudio/site-to-october
```

### From local directory

```bash
claude --plugin-dir /path/to/site-to-october
```

## Usage

Just run the command — it handles everything interactively:

```
/site-to-october:migrate https://example.com
```

The plugin will:

1. **Check your environment** — verifies curl/wget are available. If you're not in an OctoberCMS project, it offers to create one.
2. **Discover** all URLs on the site (sitemap or crawl)
3. **Scrape** one page per unique template (desktop + mobile)
4. **Download** all CSS, JS, images, and fonts
5. **Analyze** HTML to identify layout, header, footer, nav, sidebar, content areas
6. **Generate** the full OctoberCMS theme (layouts, partials, pages, assets)
7. **Ask you** how to handle each content group (Tailor blueprints, RainLab.Blog, static)
8. **Wire up** components and import content

Then keep iterating:
```
> the mobile menu isn't toggling, fix the JS
> make the hero editable via Tailor
> use RainLab.Blog for the news section
```

## What Gets Generated

```
themes/<theme-name>/
├── theme.yaml
├── layouts/
│   └── default.htm          # Main layout with {% page %}, {% partial %} tags
├── partials/
│   ├── header.htm            # Site header + logo
│   ├── footer.htm            # Site footer
│   ├── nav.htm               # Navigation menu
│   └── ...                   # Any other repeated blocks
├── pages/
│   ├── home.htm              # Homepage
│   ├── about.htm             # Static pages
│   ├── blog.htm              # Listing page (if applicable)
│   ├── blog-post.htm         # Detail page (if applicable)
│   └── ...
├── content/                  # Static content blocks
├── blueprints/               # Tailor blueprints (if chosen)
└── assets/
    ├── css/
    ├── js/
    ├── images/
    └── fonts/
```

## Content Strategy

The plugin doesn't auto-decide how to manage content. After analysis, it presents each content group and lets you choose:

| Content Type | Suggested Approach |
|-------------|-------------------|
| Blog / News | Full plugin (production) or Tailor stream |
| Services / Portfolio / Team | Full plugin (production) or Tailor stream |
| Homepage sections | Tailor single blueprint |
| Site settings / Social links | Tailor global blueprint |
| Static pages (About, Contact) | Inline in theme pages |

### Multilingual Support

For multilingual sites, the plugin automatically:
- Detects languages via hreflang, URL prefixes, and language switchers
- Installs `rainlab/translate-plugin` for `localeUrl` support
- Adds `localeUrl` viewBag entries to every page for translated URLs
- Configures `localePicker` and `sitePicker` components in layouts
- Uses `|page` filter for all internal links (locale-aware routing)
- Adds `|trans` to titles and user-visible strings
- Sets up `whereTranslation` fallback in components for translated slug lookup
- Creates `lang/en.json` (or other locales) with string translations

## Works With Any Website

It works on any website — WordPress, Squarespace, Wix, Shopify, custom-built, static HTML. The scraping is HTML-based, so the source CMS doesn't matter.

If the source happens to be WordPress and the REST API is available, the plugin will opportunistically grab structured content as a bonus.

## Plugin Structure

```
site-to-october/
├── .claude-plugin/
│   └── plugin.json           # Manifest + dependency on softappstudio/octobercms
├── commands/
│   └── migrate.md            # /site-to-october:migrate command (6-phase pipeline)
├── skills/
│   └── octobercms-theming/
│       └── SKILL.md          # HTML → OctoberCMS conversion patterns
└── README.md
```

## License

MIT
