# Contributing to the Ultimate Data Engineering Guide

Thank you for your interest in contributing! This guide is built for the data engineering community and every contribution — big or small — makes it better for everyone.

**Maintainer:** [Mano Harsha Sappa](https://www.linkedin.com/in/manoharshasappa/)  
**Repository:** https://github.com/ManoHarshaSappa/Ultimate-Data-Engineering-Guide

---

## What You Can Contribute

| Type | Examples |
|------|---------|
| **Fix errors** | Broken links, outdated tool versions, incorrect info |
| **Add tools** | New tools in any category with a link and one-line description |
| **Add resources** | Books, newsletters, podcasts, YouTube channels, communities |
| **Add case studies** | Company data engineering architecture deep-dives |
| **Improve explanations** | Clearer descriptions, better examples |
| **Add sections** | New topics not yet covered (e.g. GenAI + Data Engineering) |

---

## How to Contribute

### 1. Fork and clone

```bash
git clone https://github.com/<your-username>/Ultimate-Data-Engineering-Guide.git
cd Ultimate-Data-Engineering-Guide
```

### 2. Create a branch

```bash
git checkout -b add/new-tool-xyz
# or
git checkout -b fix/broken-link-section-5
```

### 3. Make your changes

- **README.md** — the main guide (all sections are here)
- **guide/** — 16 detailed deep-dive files, one per topic

### 4. Commit and push

```bash
git add README.md
git commit -m "Add ClickHouse to OLAP databases section"
git push origin add/new-tool-xyz
```

### 5. Open a Pull Request

Go to the repository on GitHub and open a Pull Request against `main`. Describe what you changed and why.

---

## Contribution Guidelines

### Adding a Tool

Use this format when adding a tool to the Tools Catalog or any section:

```markdown
| [Tool Name](https://tool-website.com) | One-line description of what it does |
```

- Include the official website link
- Keep descriptions neutral and factual (not marketing copy)
- Place it in the correct category

### Adding a Book / Resource

```markdown
| [Book Title — Author](https://link) | Why it's valuable (one sentence) |
```

### Adding a Community / Newsletter

Add to Section 26 (Newsletters), 27 (Podcasts), 28 (YouTube), 29 (LinkedIn), or 30 (Communities) with a direct link.

### Adding a Case Study

Follow the format in Section 20 — include:
- Company name + scale (data volume, users, etc.)
- Key tools they built or popularized
- Link to their engineering blog

---

## Quality Bar

Before submitting, please check:

- [ ] Links are valid and go to the official source
- [ ] No duplicate entries (search the README first)
- [ ] Correct section placement
- [ ] Neutral, factual language — no promotional copy
- [ ] For new sections: follows the existing heading format (`## N. Section Name`)

---

## What We Don't Accept

- Paid promotions or sponsored content
- Tools without a public website or documentation
- Duplicate entries
- Off-topic content (this guide is specific to data engineering)

---

## Reporting Issues

Found a broken link, outdated info, or incorrect content? Please [open an issue](https://github.com/ManoHarshaSappa/Ultimate-Data-Engineering-Guide/issues) with:

- Section name and line number
- What's wrong
- What the correct information should be

---

## Questions?

Reach out on [LinkedIn](https://www.linkedin.com/in/manoharshasappa/) or open an issue on GitHub.

---

*Thank you for helping make this the best data engineering reference on the internet!*
