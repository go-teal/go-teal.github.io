# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Teal Documentation Website** - a Hugo-based static site for documenting Teal, a high-performance ETL tool built on Go. The site features a custom theme with WCAG 2.2 compliant colors, dynamic sidebar navigation, and Mermaid diagram support.

## Development Commands

### Start Development Server
```bash
hugo server
# Site runs at http://localhost:1313/
# Auto-reloads on file changes
```

### Build for Production
```bash
hugo
# Outputs to public/ directory
```

### Clean Rebuild
```bash
rm -rf public && hugo
# Use when CSS/template changes don't appear
```

### Force Full Rebuild (disable fast render)
```bash
hugo server --disableFastRender
```

## Architecture

### Hugo Site Structure

The site uses **Hugo's single-page layout with dynamic sidebars**:

- **Homepage** (`layouts/index.html`): Hero section, feature cards, quick example
- **Documentation pages** (`layouts/_default/single.html`): Sidebar + content layout
- **Sidebar** (`layouts/partials/sidebar.html`): Context-aware navigation that shows different menus based on URL path
- **Base template** (`layouts/_default/baseof.html`): Wraps all pages, includes header/footer/scripts

### Content Organization

All content is in **single Markdown files** (not directories):
- `content/_index.md` - Homepage (minimal, most content in index.html template)
- `content/docs.md` - Complete documentation in ONE file
- `content/api.md` - API reference in ONE file
- `content/quickstart.md` - Quick start guide in ONE file

### Menu System (hugo.toml)

Three separate menu systems defined in `hugo.toml`:
1. **`[[menus.main]]`** - Top navigation (Home, Quick Start, Documentation, API, GitHub)
2. **`[[menus.docs]]`** - Sidebar for /docs/ page (uses anchor links like `#configyaml`)
3. **`[[menus.api]]`** - Sidebar for /api/ page
4. **`[[menus.quickstart]]`** - Sidebar for /quickstart/ page

The sidebar (`layouts/partials/sidebar.html`) **automatically detects the current URL** and shows the appropriate menu. Parent-child relationships use `parent = 'Parent Name'`.

### CSS Architecture

**Location**: `themes/teal-theme/static/css/teal.css`

**Color System** (WCAG 2.2 AA compliant):
```css
--color-blue: #05668d;        /* Links, interactive elements (4.51:1) */
--color-gray: #1a1a1a;        /* Text (16.1:1) */
--color-green: #2d5016;       /* Success states (8.2:1) */
--color-red: #991b1b;         /* Error states (7.1:1) */
--color-white: #ffffff;       /* Backgrounds */
--color-gray-light: #d1ecf8;  /* Hover states, table alternating rows */
```

**Key Rules**:
- **NO hardcoded colors** - always use CSS variables
- All backgrounds are white (`var(--color-white)`)
- Sidebar uses `var(--color-gray-light)` for hover states
- Table headers use blue background with white text
- Feature cards use light gray background with no borders

**Layout**:
- `.docs-layout` - Flexbox container for sidebar + content
- `.sidebar` - 240px fixed width, sticky positioning
- `.content-wrapper` - `flex: 1` (takes remaining space, NO max-width)
- Sidebar menu items have minimal padding (`0.25rem` vertical, `0.5rem` horizontal for parents, `1.5rem` left for children)

### Mermaid Diagram Support

**Implementation**: `layouts/_default/baseof.html` (bottom of file)

Hugo renders mermaid code blocks as `<code class="language-mermaid">`. JavaScript converts these to `<div class="mermaid">` on page load:

```javascript
document.querySelectorAll('code.language-mermaid').forEach(function(code) {
    const div = document.createElement('div');
    div.className = 'mermaid';
    div.textContent = code.textContent;
    code.parentElement.replaceWith(div);
});
mermaid.run();
```

Uses **regular script tag** (not ES module) to avoid CSP errors.

## Common Modifications

### Adding New Menu Items

Edit `hugo.toml`:
```toml
[[menus.docs]]
  name = 'New Section'
  pageRef = '/docs#new-section'
  weight = 9

[[menus.docs]]
  name = 'Subsection'
  pageRef = '/docs#subsection'
  parent = 'New Section'  # Creates parent-child relationship
  weight = 1
```

### Updating Colors

All colors are defined in `:root` at the top of `themes/teal-theme/static/css/teal.css`. Change the hex values there. **Never hardcode colors in the CSS** - always use `var(--color-name)`.

### Adding New Content Sections

Add to the appropriate single markdown file (`docs.md`, `api.md`, or `quickstart.md`) using H2 headers (`##`). The `id` is auto-generated from the header text (lowercase, hyphens for spaces).

### Modifying Sidebar Behavior

The sidebar detection logic is in `layouts/partials/sidebar.html`:
```go
{{ $menuID := "main" }}
{{ if hasPrefix $currentURL "/docs/" }}
    {{ $menuID = "docs" }}
{{ else if hasPrefix $currentURL "/api/" }}
    {{ $menuID = "api" }}
```

## Important Notes

- **Browser caching**: CSS changes often require hard refresh (Ctrl+Shift+R)
- **Hugo server port conflicts**: Hugo auto-increments port if 1313 is busy (check terminal output)
- **Sidebar styling**: Parent items are bold (`font-weight: 600`), children are indented with left padding
- **Favicon**: Located at `themes/teal-theme/static/favicon.ico` and `favicon.svg`
- **No hover effects**: Feature cards, code blocks, and sidebar items have hover disabled per design requirements
- **Full-width content**: Content wrapper has no max-width to use full available space

## Hugo Template System

- **`{{ define "main" }}`** - Content blocks in single.html, index.html
- **`{{ partial "name.html" . }}`** - Include partials (pass context with `.`)
- **`{{ .Content }}`** - Rendered markdown content
- **`{{ .Title }}`** - Page title from frontmatter
- **`{{ .Site.Params.variable }}`** - Access hugo.toml params
- **`{{ or .URL .PageRef }}`** - Fallback pattern for menu links (use pageRef when URL empty)

## Deployment

The site is configured for GitHub Pages with `baseURL = 'https://go-teal.github.io/'`. Build with `hugo` and deploy the `public/` directory.
