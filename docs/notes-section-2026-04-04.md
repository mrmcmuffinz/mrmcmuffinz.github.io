# Notes Section: Short-Form Microblog

**Date:** 2026-04-04
**Author:** Abraham Cabrera (with Claude Code)

---

## Overview

Added a `/notes/` section to the Hugo site for short-form updates — study progress, quick observations, and thoughts that don't warrant a full blog post. The section has its own RSS feed and navigation entry, and recent notes appear on the homepage.

```mermaid
graph LR
    A[Write a note] -->|git push| B[Hugo Build]
    B --> C[/notes/ list page]
    B --> D[/notes/index.xml RSS]
    B --> E[Homepage 'Recent Notes']
```

---

## What Changed

### New Content Section

Notes live in `content/notes/` with minimal frontmatter (title, date, optional tags). The section index `_index.md` uses Hugo's `cascade` feature to apply settings to all notes automatically:

- `layout: simple` — Blowfish's built-in minimal layout (title + content only)
- Reading time, word count, and comments disabled
- List page groups notes by year with summaries shown inline

### Homepage Addition

A custom `layouts/partials/home/profile.html` overrides the Blowfish theme partial to add a "Recent Notes" section below the existing "Recent Articles" section. Shows the 3 most recent notes as a compact date + title list with a "View more" link to `/notes/`.

### Navigation

"Notes" added to the main navigation bar (weight 20, appears before Privacy and Contact).

### What Was NOT Changed

- `mainSections` remains `["posts"]` — notes are excluded from the "Recent Articles" homepage section
- LinkedIn RSS workflow continues to watch `/posts/index.xml` only
- No custom templates beyond the homepage override — Blowfish's `simple.html` and `list.html` handle individual notes and the list page

---

## Architecture

```mermaid
graph TD
    subgraph Content
        A[content/notes/_index.md<br/>Section config + cascade] --> B[content/notes/*.md<br/>Individual notes]
    end

    subgraph Layouts
        C[layouts/partials/home/profile.html<br/>Homepage override with Recent Notes]
        D[themes/blowfish simple.html<br/>Individual note layout]
        E[themes/blowfish list.html<br/>Notes list page layout]
    end

    subgraph Config
        F[menus.en.toml<br/>Notes nav entry]
    end

    subgraph Output
        G[/notes/ — list page]
        H[/notes/slug/ — individual note]
        I[/notes/index.xml — RSS feed]
        J[Homepage — Recent Notes section]
    end

    B --> D --> H
    A --> E --> G
    B --> I
    C --> J
    F --> G
```

---

## Implementation Details

### `content/notes/_index.md`

```toml
+++
title = "Notes"
description = "Short-form thoughts, links, and observations."
groupByYear = true
showSummary = true

[cascade]
  layout = "simple"
  showReadingTime = false
  showWordCount = false
  showDate = true
  showAuthor = false
  showComments = false
  showPagination = false
+++
```

### `content/notes/first-note.md` (example)

```toml
+++
date = '2026-04-04T12:00:00-05:00'
title = 'Testing the notes section'
tags = ["meta"]
+++
```

Just a quick test of the new microblog section. Short thoughts go here.

### Menu entry addition (`config/_default/menus.en.toml`)

```toml
[[main]]
  name = "Notes"
  pageRef = "notes"
  weight = 20
```

### Homepage override (`layouts/partials/home/profile.html`)

Copy `themes/blowfish/layouts/partials/home/profile.html` verbatim, then append a "Recent Notes" section after the existing `<section>` that calls `recent-articles/main.html`. The new section should:

1. Query recent notes: `range first 3 (where .Site.RegularPages "Section" "notes")`
2. Render each as a compact list item: date (`dateFormat "Jan 2, 2006"`) + linked title
3. Include a "View more" button linking to `/notes/` using the same styling as the existing "Show more" button in `recent-articles/main.html`
4. Wrap in a heading: "Recent Notes" (use `<h2>` with same classes as the "Recent Articles" heading: `mt-8 text-2xl font-extrabold mb-10`)

---

## Files Changed

| Action | File | Purpose |
|--------|------|---------|
| Create | `content/notes/_index.md` | Section index with cascade config |
| Create | `content/notes/first-note.md` | Example note |
| Create | `layouts/partials/home/profile.html` | Homepage override adding Recent Notes |
| Modify | `config/_default/menus.en.toml` | Add Notes to main nav |

---

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| Use Blowfish's `simple.html` layout via cascade | No custom templates needed for individual notes; minimal display fits short-form content |
| Individual pages per note (not list-only) | Each note gets a shareable URL |
| Homepage override instead of modifying theme | Hugo lookup order means project `layouts/` takes precedence; theme submodule stays untouched |
| Separate from `mainSections` | Notes don't dilute the blog's signal; homepage recent posts remain blog-only |
| No cross-posting to LinkedIn (yet) | Evaluate after consistent usage for 1-2 months before adding syndication complexity |

---

## Future Considerations

- **Cross-posting:** If notes are used consistently, add GitHub Actions workflows to syndicate to LinkedIn/Facebook via their APIs
- **Tagging:** Notes support tags already; could add a dedicated taxonomy page if volume warrants it
- **Removal:** If the section goes unused, it's just a content directory and a homepage partial to remove — low cost to revert
