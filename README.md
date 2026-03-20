# Chandra's Ckloud Journal

Personal technical blog documenting hands-on cloud projects across AWS and Azure.

**Live site:** https://ckanth15.github.io/ckloud-journal/

---

## Tech Stack

- **Static site generator:** Hugo v0.158.0
- **Theme:** [Blowfish](https://blowfish.page/) (Tailwind CSS)
- **Hosting:** GitHub Pages
- **Deploy:** GitHub Actions (auto-deploys on push to `main`)
- **Repo:** https://github.com/ckanth15/ckloud-journal

## Environments

| Machine | Role | Hugo installed? |
|---------|------|----------------|
| Ubuntu machine | Preview & authoring | Yes (snap) |
| Mac | Edit & push | No |

**Preview locally on Ubuntu:**
```bash
cd ~/ckloud-journal
hugo server -D --bind 0.0.0.0
# Open http://192.168.x.x:1313/ckloud-journal/ from Mac (use your Ubuntu machine's local IP)
```

---

## GitHub Actions & Deployment

The site auto-deploys via GitHub Actions whenever you push to `main`.

- **Workflow file:** `.github/workflows/hugo.yml`
- **Actions dashboard:** https://github.com/ckanth15/ckloud-journal/actions
- **GitHub Pages settings:** https://github.com/ckanth15/ckloud-journal/settings/pages

### How it works

1. Push to `main` triggers the workflow
2. GitHub Actions checks out the repo (with submodules for the Blowfish theme)
3. Hugo builds the site
4. Output is deployed to GitHub Pages

### Workflow file (.github/workflows/hugo.yml)

```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.158.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: America/Los_Angeles
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### Troubleshooting deployments

- Check build status: https://github.com/ckanth15/ckloud-journal/actions
- Green checkmark = deployed successfully
- If a build fails, click into the run to see the error logs
- Common issues: missing submodule (theme not found), invalid TOML config

---

## Adding a New Blog Post

### 1. Create a page bundle

```bash
mkdir -p content/posts/my-new-post
```

### 2. Add index.md

```bash
cat > content/posts/my-new-post/index.md << 'EOF'
---
title: "My Post Title"
date: 2026-03-20T10:00:00-07:00
description: "Short description for cards and SEO."
tags: ["AWS", "Lambda", "Serverless"]
author: "Chandra Kanth"
draft: false
---

Post content goes here...
EOF
```

### 3. Add a feature image (optional but recommended)

Drop any image starting with `feature` into the post folder:

```
content/posts/my-new-post/
├── index.md
└── feature-architecture.png   # Auto-detected as card thumbnail + hero image
```

### 4. Add inline screenshots

Place images in the same folder and reference them in markdown:

```markdown
![Description](screenshot-name.png)
```

### 5. Preview and push

```bash
# Preview on Ubuntu machine
hugo server -D --bind 0.0.0.0

# Push from either machine
git add -A
git commit -m "Add post: my-new-post"
git push origin main
```

---

## Config Structure

```
config/_default/
├── hugo.toml            # Base URL, theme, markup settings
├── languages.en.toml    # Site title, author name, social links
├── menus.en.toml        # Nav bar items (Home, Blog, Tags, About)
└── params.toml          # Theme settings (layout, colors, article options)
```

### Key settings in params.toml

| Setting | Current Value | What it does |
|---------|--------------|--------------|
| `colorScheme` | `ocean` | Theme colors (options: ocean, congo, avocado, fire, slate, etc.) |
| `defaultAppearance` | `dark` | Default light/dark mode |
| `enableSearch` | `true` | Search bar in nav |
| `enableCodeCopy` | `true` | Copy button on code blocks |
| `[homepage] layout` | `card` | Homepage style (options: card, profile, hero, page, background, custom) |
| `[article] showHero` | `true` | Show feature image inside posts |
| `[article] heroStyle` | `big` | Hero image style (options: basic, big, background, thumbAndBackground) |

### Social links (in languages.en.toml)

```toml
[params.author]
  name = "Chandra Kanth"
  links = [
    { linkedin = "https://www.linkedin.com/in/chandrakanthsingh/" },
    { github = "https://github.com/ckanth15" },
  ]
```

---

## Content Structure

```
content/
├── _index.md                              # Homepage (keep title empty)
├── about.md                               # About page
└── posts/
    ├── opensearch-search-engine/
    │   ├── index.md                       # Blog post
    │   └── feature-architecture.png       # Card thumbnail + hero
    └── lambda-rds-mysql-debugging/
        ├── index.md                       # Blog post
        └── feature-lambda.png             # Card thumbnail + hero
```

---

## Blog Writing Guidelines

- **Vendor-neutral:** Never mention lab provider names 
- **Use "learner"** instead of "student"
- **First person, concise** — practical tone, not AI-sounding
- **Tags** — use specific tech tags: AWS, Azure, Lambda, OpenSearch, Serverless, CI/CD, etc.

---

## Useful Commands

```bash
# Preview with drafts
hugo server -D --bind 0.0.0.0

# Build site (usually handled by GitHub Actions)
hugo --minify

# Check theme version
cat themes/blowfish/package.json | grep version

# Update Blowfish theme
cd themes/blowfish && git pull origin main && cd ../..
```

---

## GitHub Token

- Type: Fine-grained, scoped to ckloud-journal repo
- Expires: Jun 17, 2026
- Mac credential helper: osxkeychain
