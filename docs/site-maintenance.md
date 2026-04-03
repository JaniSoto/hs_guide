# Maintaining This Site

How to edit the guide, preview changes, and deploy to GitHub Pages.

---

## How the Site Works

This guide is built with **MkDocs Material** — you write pages in plain Markdown (`.md` files), and MkDocs turns them into a beautiful static website. The site is hosted on **GitHub Pages** for free.

Your workflow:
```
Edit .md files → git push → mkdocs gh-deploy → site updates
```

---

## One-Time Setup

Do this once on any computer where you want to edit the guide.

**1. Clone the repository:**

```bash
git clone https://github.com/JaniSoto/hs_guide.git
cd hs_guide
```

**2. Create a Python virtual environment (the "bubble"):**

```bash
python3 -m venv .venv
source .venv/bin/activate   # Linux/macOS
# Windows: .venv\Scripts\activate
```

**3. Install MkDocs Material:**

```bash
pip install mkdocs-material
```

The `.gitignore` file in the repo already excludes `.venv/` so it will never be accidentally pushed to GitHub.

---

## Editing the Guide

All content lives in the `docs/` folder as `.md` files. Open any file in any text editor.

**File structure:**

```
docs/
├── index.md                    ← Homepage
├── before-you-begin.md
├── assets/
│   └── extra.css               ← All visual styling lives here
├── phases/
│   ├── phase-0-install.md
│   ├── phase-1-first-boot.md
│   └── ...
└── reference/
    ├── quick-commands.md
    ├── troubleshooting.md
    └── ...
```

**To add a new page:**

1. Create a new `.md` file in the appropriate folder
2. Add it to the `nav:` section in `mkdocs.yml`
3. Preview and deploy (see below)

**To add a new phase:** Copy an existing phase file as a template. The phase header HTML at the top of each page can be reused — just change the number and text.

---

## Preview Locally

Before deploying, preview your changes in the browser with live reload:

```bash
source .venv/bin/activate   # If not already active
mkdocs serve
```

Open `http://localhost:8000` in your browser. Changes to `.md` files or `extra.css` update the preview automatically.

---

## Deploy to GitHub Pages

When you're happy with the changes:

**Step 1 — Push your Markdown and config to GitHub:**

```bash
git add .
git status   # Check what's being committed — should only show .md files and mkdocs.yml
git commit -m "Your description of changes"
git push origin main
```

!!! warning "Check `git status` before committing"
    If `git status` shows thousands of files inside `.venv/`, stop — your `.gitignore` is missing. Create it: `echo ".venv/" > .gitignore` and commit that file first.

**Step 2 — Build and deploy the website:**

```bash
mkdocs gh-deploy
```

This builds the static HTML and pushes it to the `gh-pages` branch of your repo. GitHub Pages automatically serves it at `https://janisoto.github.io/hs_guide/` within a minute or two.

---

## Common Edits

**Change a code block:** Find the fenced code block in the relevant `.md` file and edit the text between the triple backticks.

**Change admonition text** (the warning/tip/note boxes): Find the `!!! warning "Title"` block and edit the indented text below it.

**Add a tip box:**

```markdown
!!! tip "Title of the tip"
    Your tip text here. Must be indented with 4 spaces.
```

**Change colours or fonts:** Edit `docs/assets/extra.css`. The colour tokens are at the top of the file under `[data-md-color-scheme="slate"]` (dark mode) and `[data-md-color-scheme="default"]` (light mode).

**Reorder navigation:** Edit the `nav:` section in `mkdocs.yml`. The indentation determines the hierarchy.

---

## MkDocs Material Documentation

For a full reference of all available features (admonitions, tabs, icons, etc.):

**[squidfunk.github.io/mkdocs-material](https://squidfunk.github.io/mkdocs-material/)**
