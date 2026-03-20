# Chandra's Ckloud Journal

Personal technical blog documenting hands-on cloud projects across AWS and Azure.

**Live site:** https://ckanth15.github.io/ckloud-journal/

---

## Tech Stack

- **Static site generator:** Hugo v0.158.0
- **Theme:** [Blowfish](https://blowfish.page/) (Tailwind CSS)
- **Hosting:** GitHub Pages
- **Deploy:** GitHub Actions (auto-deploys on push to `main`)

## Environments

| Machine | Role | Hugo installed? | Path |
|---------|------|----------------|------|
| linkstationu (Ubuntu, 192.168.1.6) | Preview & authoring | Yes (snap) | `~/ckloud-journal` |
| Mac | Edit & push | No | `~/ckrepo/ckloud-journal` |

**Preview locally on linkstationu:**
```bash
cd ~/ckloud-journal
hugo server -D --bind 0.0.0.0
# Open http://192.168.1.6:1313/ckloud-journal/ from Mac
```

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
# Preview on linkstationu
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

- **Vendor-neutral:** Never mention lab provider names (e.g., no Whizlabs, etc.)
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
