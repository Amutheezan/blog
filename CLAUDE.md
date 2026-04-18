# CLAUDE.md — Codebase Guide for AI Assistants

This is the personal academic portfolio website for Amutheezan Sivagnanam, a Postdoctoral Fellow at the University of Houston researching AI for Transportation Optimization and Reinforcement Learning. The site is deployed at https://amutheezan.com.

---

## Technology Stacks

- **Static site generator:** Jekyll (via the `github-pages` gem)
- **Theme:** Minimal Mistakes (v3.4.2, vendored in-repo)
- **Templating:** Liquid
- **Stylesheets:** SCSS (compiled by Jekyll)
- **JavaScript:** jQuery + plugins, minified via uglify-js
- **Hosting:** GitHub Pages with custom domain (`amutheezan.com`, see `CNAME`)
- **Comments:** Staticman (configured but inactive)

---

## Directory Structure

```
.
├── _config.yml            # Main Jekyll configuration
├── _config.dev.yml        # Development overrides (local URL, no analytics)
├── Gemfile                # Ruby gem dependencies (Jekyll, hawkins)
├── package.json           # Node.js build scripts (JS minification)
├── CNAME                  # Custom domain: amutheezan.com
│
├── _data/
│   ├── navigation.yml     # Top-nav menu items
│   ├── authors.yml        # Author metadata (bio, links, avatar)
│   └── ui-text.yml        # Localized UI strings for the theme
│
├── _layouts/              # Liquid HTML layout templates
│   ├── default.html       # Base layout (includes head, masthead, footer)
│   ├── single.html        # General page/post layout
│   ├── talk.html          # Talk-specific layout
│   ├── splash.html        # Hero/splash page layout
│   ├── archive.html       # Archive/listing page layout
│   └── compress.html      # HTML minification layout (wraps default)
│
├── _includes/             # Reusable Liquid partials (~36 files)
│   ├── head/              # <head> section includes
│   ├── footer/            # Footer includes
│   ├── masthead.html      # Top navigation bar
│   ├── author-profile.html# Sidebar author bio
│   ├── archive-single.html# Single entry in archive/listing views
│   └── ...
│
├── _sass/                 # SCSS source modules
│   ├── _variables.scss    # Colors, typography, breakpoints
│   ├── _base.scss         # Base element styles
│   ├── _page.scss         # Page/post layout styles
│   ├── _archive.scss      # Archive/listing styles
│   ├── _sidebar.scss      # Sidebar styles
│   ├── _navigation.scss   # Nav/masthead styles
│   ├── _syntax.scss       # Code block syntax highlighting
│   └── vendor/            # Third-party SCSS (Breakpoint, Susy, Font Awesome, Magnific Popup)
│
├── assets/
│   ├── css/
│   │   ├── main.scss      # CSS entry point (imports all _sass modules)
│   │   └── academicons.css# Academic icon set
│   └── js/
│       ├── _main.js       # Custom jQuery initialization (source)
│       ├── main.min.js    # Minified bundle (DO NOT edit directly)
│       ├── plugins/       # jQuery plugins (source)
│       └── vendor/        # Vendor JS (jQuery 1.12.4)
│
├── _pages/                # Static site pages
│   ├── about.md           # Home page (permalink: /)
│   ├── cv.md              # Full curriculum vitae
│   ├── publications.html  # Publications archive page
│   ├── talks.html         # Talks archive page
│   ├── workshops.html     # Workshops archive page
│   ├── projects.html      # Projects archive page
│   └── 404.md             # 404 error page
│
├── _publications/         # Research publication entries (6 items)
├── _talks/                # Conference talk entries (3 items)
├── _workshops/            # Workshop entries (1 item)
├── _projects/             # Portfolio project entries (6 items)
├── _posts/                # Blog posts (~40 entries, 2012–2016)
├── _drafts/               # Unpublished draft posts
│
├── files/                 # Downloadable assets
│   ├── CurriculumVitae.pdf
│   ├── MLE_Resume.pdf
│   ├── DS_Resume.pdf
│   ├── SE_Resume.pdf
│   ├── Presentations/     # PDF slides from talks
│   └── Posters/           # Conference poster PDFs
│
└── images/                # Image assets (~50+ files)
```

---

## Collections and Content Conventions

Jekyll collections are defined in `_config.yml`. Each collection item is a Markdown file with YAML frontmatter.

### Publications (`_publications/`)

File naming: `YYYY-MM-DD-short-slug.md`

Required frontmatter:
```yaml
---
title: "Full paper title"
collection: publications
permalink: /publications/slug/
date: YYYY-MM-DD
venue: "Venue name (e.g., ICML 2024)"
citation: "Full citation string"
---
```

Body: Markdown with links to paper (OpenReview/PMLR/ACM), source code, videos, posters, slides. Use standard Markdown links — no custom shortcodes.

### Talks (`_talks/`)

File naming: `YYYY-MM-DD-short-slug.md`

Required frontmatter:
```yaml
---
title: "Talk title"
collection: talks
type: "Talk"                # or "Poster", "Workshop"
permalink: /talks/slug/
date: YYYY-MM-DD
venue: "Conference Name"
location: "City, Country"
---
```

Body: Links to slides (PDF in `files/Presentations/`), posters (PDF in `files/Posters/`), paper, code.

### Projects (`_projects/`)

File naming: `YYYY-MM-DD-short-slug.md`

Frontmatter fields: `title`, `collection: projects`, `permalink`, `date`. Body describes the project.

### Blog Posts (`_posts/`)

File naming: `YYYY-MM-DD-slug.md`

Frontmatter:
```yaml
---
type: post
title: "Post Title"
author: amutheezan
category: [category1]
date: YYYY-MM-DD HH:MM:SS
last_modified_at: YYYY-MM-DD HH:MM:SS
tags: [tag1, tag2]
---
```

Existing posts cover technical tutorials from 2012–2016 (HL7, WSO2, Arduino, RFID, healthcare IT).

---

## Key Configuration: `_config.yml`

Important settings to be aware of:

- `url: https://amutheezan.com` — production URL
- `baseurl: ""` — no subdirectory; all paths are root-relative
- Collections: `publications`, `talks`, `workshops`, `projects` (all with `output: true`)
- Default layouts: posts → `single`, talks → `talk`, all with `author_profile: true`
- Plugins: `jekyll-paginate`, `jekyll-sitemap`, `jekyll-gist`, `jekyll-feed`, `jekyll-redirect-from`
- SASS: `compressed` style in production; `_config.dev.yml` overrides to `expanded`

Navigation is controlled by `_data/navigation.yml`. Author info (bio, avatar, social links) is in `_data/authors.yml`.

---

## Local Development

### Prerequisites

- Ruby + Bundler
- Node.js + npm

### Setup

```bash
bundle install    # Install Jekyll and Ruby gems
npm install       # Install JS build tools (uglify-js, onchange)
```

### Running the dev server

```bash
# Standard (uses production config)
bundle exec jekyll serve

# With development overrides (local URL, no analytics, expanded SASS)
bundle exec jekyll serve --config _config.yml,_config.dev.yml

# With live reload (hawkins gem)
bundle exec hawkins --config _config.yml,_config.dev.yml
```

Site is available at `http://localhost:4000`.

### JavaScript build

Only needed when editing JS source files in `assets/js/plugins/` or `assets/js/_main.js`:

```bash
npm run build:js    # Minify all JS into assets/js/main.min.js
npm run watch:js    # Auto-rebuild on changes
```

**Important:** Never edit `assets/js/main.min.js` directly — it is generated by the build script.

---

## Styling Conventions

- SCSS variables (colors, fonts, breakpoints) are defined in `_sass/_variables.scss`. Modify there for theme-wide changes.
- The CSS entry point is `assets/css/main.scss` which imports all modules via `@import`.
- Vendor SCSS lives in `_sass/vendor/` — do not modify vendor files.
- The site uses Font Awesome (via SCSS) and Academicons (via `assets/css/academicons.css`) for icons.
- HTML is compressed via the `compress` layout wrapping `default` — avoid inline `<script>` blocks with `//` JS comments (compression strips newlines and breaks them).

---

## Deployment

- **Automatic:** Push to `master` → GitHub Pages rebuilds and publishes.
- **Custom domain:** Controlled by `CNAME` file (`amutheezan.com`). Do not delete or modify this file.
- **No CI pipeline** — GitHub Pages handles the build.

---

## Common Tasks

### Add a new publication

1. Create `_publications/YYYY-MM-DD-slug.md` with required frontmatter (see above).
2. Optionally add PDF/slides to `files/Presentations/` or `files/Posters/`.
3. Update `_pages/cv.md` to include the publication in the CV section.

### Add a new talk

1. Create `_talks/YYYY-MM-DD-slug.md` with required frontmatter.
2. Add slides PDF to `files/Presentations/` and poster PDF to `files/Posters/` if applicable.

### Update the CV

Edit `_pages/cv.md` directly. It is a plain Markdown file — no special data source. Keep the existing formatting conventions (section headers, bullet lists).

### Update author bio or social links

Edit `_data/authors.yml`. The sidebar profile pulls data from this file. Fields include `name`, `bio`, `location`, `email`, `uri`, and social links (`github`, `linkedin`, `twitter`, `google_scholar`, `orcid`, `researchgate`).

### Add/remove navigation items

Edit `_data/navigation.yml`. The `main` key controls the top navigation bar.

### Update downloadable files (CV PDF, resumes)

Replace files directly in the `files/` directory. The `_pages/cv.md` links to these paths.

---

## What NOT to do

- Do not edit `assets/js/main.min.js` directly — it is auto-generated.
- Do not delete `CNAME` — it controls the custom domain.
- Do not modify files in `_sass/vendor/` — these are vendored third-party libraries.
- Do not add `---` YAML frontmatter to files in `assets/` — Jekyll will try to process them.
- Avoid inline JS `//` line comments inside `<script>` tags in layouts/includes (HTML compression removes newlines and will break the JS).
