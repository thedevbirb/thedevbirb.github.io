# Blog

A technical blog. Written in [Obsidian](https://obsidian.md), published with Jekyll on GitHub Pages.

## Setup

1. Open this folder as an Obsidian vault.
2. Write posts in `_posts/` named `YYYY-MM-DD-slug.md` (front matter auto-added by the template in `templates/`).
3. Commit and push to GitHub — Pages builds and hosts automatically.

## Publishing a new note

### 1. Create the post in Obsidian

Insert the template: Obsidian → Templates → `post` (`Ctrl/Cmd+P` → "Templates: Insert template").

Fill in the front matter and body. The template creates a file in `_posts/` named `YYYY-MM-DD-slug.md`, e.g. `2026-08-24-my-new-post.md`.

### 2. Set the date

Make sure the `date` field in the front matter is correct. Jekyll only publishes posts with a date in the past — a future date means the post won't appear.

### 3. Preview locally (optional)

From the repo root:

```bash
bundle exec jekyll serve
```

Then open `http://localhost:4000`. Your post should appear on the home page and at `/YYYY/MM/DD/slug.html`.

### 4. Commit and push

```bash
git add _posts/
git commit -m "Add post: <title>"
git push origin main
```

GitHub Actions builds the site and deploys it to Pages automatically. Once the workflow finishes, the post is live.

## Troubleshooting

- **Post isn't showing up** — the `date` is in the future, or the filename doesn't follow `YYYY-MM-DD-slug.md`.
- **Build fails on CI** — run `bundle exec jekyll build` locally and check for errors; fix any Liquid/syntax issues before pushing.
