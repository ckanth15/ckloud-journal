# Chandra's Ckloud Journal

Personal tech blog documenting cloud computing projects, debugging stories, and certification prep.

**Live at**: https://ckanth15.github.io/ckloud-journal/

## How to Add a New Blog Post

### Option 1: From Terminal
```bash
cd ~/ckloud-journal
hugo new posts/your-post-title.md
# edit the post with your preferred editor
git add .
git commit -m "New post: Your Post Title"
git push
```

### Option 2: From GitHub Web (No Terminal Needed)

1. Go to https://github.com/ckanth15/ckloud-journal
2. Navigate to `content/posts/`
3. Click **Add file** > **Create new file**
4. Name it `your-post-title.md`
5. Add frontmatter + content (see template below)
6. Click **Commit changes**
7. Auto-deploys in ~20 seconds

### Post Template
```markdown
---
title: "Your Post Title"
date: 2026-03-19
description: "Brief description"
tags: ["AWS", "Azure"]
categories: ["AWS Projects"]
ShowToc: true
TocOpen: false
---

Your content here in Markdown...
```

## Tech Stack

- **Hugo** (v0.158.0) — Static site generator
- **PaperMod** — Blog theme
- **GitHub Pages** — Free hosting
- **GitHub Actions** — Auto-deploy on push
