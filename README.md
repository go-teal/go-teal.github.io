# Teal Documentation Website

This is the official documentation website for [Teal](https://github.com/go-teal/teal), a high-performance ETL tool built on Go.

## Overview

The documentation site is built using [Hugo](https://gohugo.io/), a fast and flexible static site generator. It features:

- **Landing Page** - Overview of Teal and its key features
- **Quick Start Guide** - Step-by-step instructions to get started with Teal
- **Complete Documentation** - Comprehensive documentation covering all features
- **API Reference** - Detailed API documentation for developers
- **Custom Theme** - Beautiful theme matching the Teal UI color schema

## Prerequisites

- [Hugo](https://gohugo.io/) v0.152.0 or later
- Go 1.21+ (if installing Hugo via `go install`)

## Installation

### Install Hugo using Go

```bash
go install github.com/gohugoio/hugo@latest
```

### Alternative: Install Hugo using package manager

**Ubuntu/Debian:**
```bash
sudo apt-get install hugo
```

**macOS:**
```bash
brew install hugo
```

**Windows:**
```bash
choco install hugo
```

## Local Development

### Start the development server

```bash
hugo server
```

The site will be available at `http://localhost:1313/`

### Start with custom port and host binding

```bash
hugo server --bind 0.0.0.0 --port 8080
```

### Enable live reload

Hugo automatically watches for changes and reloads the browser. To disable fast render:

```bash
hugo server --disableFastRender
```

## Building the Site

### Build for production

```bash
hugo
```

This generates the static site in the `public/` directory.

### Build with specific base URL

```bash
hugo --baseURL "https://yourdomain.com/"
```

### Build drafts and future content

```bash
hugo --buildDrafts --buildFuture
```

## Project Structure

```
teal-docs/
├── content/              # Markdown content files
│   ├── _index.md        # Homepage content
│   ├── quickstart.md    # Quick Start guide
│   ├── docs.md          # Main documentation
│   └── api.md           # API reference
├── themes/
│   └── teal-theme/      # Custom Teal theme
│       ├── layouts/     # HTML templates
│       │   ├── _default/
│       │   │   ├── baseof.html
│       │   │   └── single.html
│       │   ├── partials/
│       │   │   ├── header.html
│       │   │   └── footer.html
│       │   └── index.html
│       └── static/
│           └── css/
│               └── teal.css  # Theme styles
├── hugo.toml            # Hugo configuration
└── README.md            # This file
```

## Customization

### Update Site Configuration

Edit `hugo.toml` to change site settings:

```toml
baseURL = 'https://go-teal.github.io/'
languageCode = 'en-us'
title = 'Teal - High-Performance ETL Tool'
theme = 'teal-theme'
```

### Modify Color Schema

The color schema matches the Teal UI. To modify colors, edit:

```
themes/teal-theme/static/css/teal.css
```

Key color variables:
- `--color-primary: #22c55e` (Light green)
- `--color-primary-dark: #15803d` (Dark green)
- `--color-background: #fefffe` (White with green tint)

### Add New Pages

Create new markdown files in the `content/` directory:

```bash
hugo new content/new-page.md
```

Then edit the file with your content.

### Update Navigation

Edit the menu in `hugo.toml`:

```toml
[menu]
  [[menu.main]]
    name = 'New Page'
    url = '/new-page/'
    weight = 5
```

## Deployment

### Deploy to GitHub Pages

1. Build the site:
   ```bash
   hugo
   ```

2. Push the `public/` directory to the `gh-pages` branch

Or use GitHub Actions (see `.github/workflows/hugo.yml` for an example workflow)

### Deploy to Netlify

1. Connect your GitHub repository to Netlify
2. Set build command: `hugo`
3. Set publish directory: `public`

### Deploy to Vercel

1. Connect your GitHub repository to Vercel
2. Vercel will automatically detect Hugo and configure build settings

## Contributing

To contribute to the documentation:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test locally with `hugo server`
5. Submit a pull request

## Theme Features

The custom Teal theme includes:

- ✅ Responsive design for mobile and desktop
- ✅ Syntax highlighting for code blocks
- ✅ Clean, modern UI matching Teal's brand
- ✅ Fast page load times
- ✅ SEO-friendly HTML structure
- ✅ Dark mode ready (CSS variables)

## Color Schema

The theme uses the same color palette as the Teal UI:

| Color | Hex | Usage |
|-------|-----|-------|
| Primary Green | `#22c55e` | Buttons, links, brand elements |
| Dark Green | `#15803d` | Text, hover states |
| Light Green | `#dcfce7` | Backgrounds, highlights |
| Neutral Gray | `#6b7280` | Secondary text |
| Background | `#fefffe` | Page background |

## Support

For issues or questions:

- **GitHub Issues**: [go-teal/teal/issues](https://github.com/go-teal/teal/issues)
- **Email**: boris109@gmail.com
- **LinkedIn**: [Boris Ershov](https://www.linkedin.com/in/boris-ershov-2a4b9963/)

## License

This documentation is part of the Teal project. See the main [Teal repository](https://github.com/go-teal/teal) for license information.
