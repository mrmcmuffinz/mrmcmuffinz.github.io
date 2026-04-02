---
name: blog-post-writer
description: >
  Use this skill whenever writing, drafting, editing, or reviewing a blog post
  for the Hugo/Blowfish personal site. Trigger when the user asks to: write a
  new blog post, draft a post, revise a post, add screenshots to a post, or
  review a post for tone and style. Also trigger when the user says anything
  like "new post", "blog post", "write up", "write about", or references their
  Hugo site or GitHub Pages blog. This skill defines the voice, structure,
  formatting rules, and Hugo/Blowfish conventions for all blog content. It
  takes priority over general writing advice. Always read it before producing
  any blog post content, even for quick drafts.
---

# Blog Post Writer Skill

## Scope

All Markdown files in `content/posts/` of the Hugo site.

---

## Site Structure

```
content/posts/<post_slug>.md          <- Blog post (flat file, not page bundle)
static/img/<post_slug>/              <- Post images (if any)
```

Posts are flat `.md` files, not Hugo page bundles. Images go in
`static/img/<post_slug>/` and are referenced with absolute paths:
`/img/<post_slug>/image_name.png`.

The `<post_slug>` used for the image directory must match the post filename
without the `.md` extension. Example: post file `aws_ai_foundation_cert.md`
uses image directory `static/img/aws_ai_foundation_cert/`.

---

## Hugo Front Matter

All posts use TOML front matter (`+++` delimiters). Required fields:

```toml
+++
date = '2026-04-01T10:47:53-05:00'
draft = true
title = 'Post Title Here'
tags = ["tag1", "tag2"]
categories = ["category"]
description = "One-sentence summary for metadata and previews."
layout = "post"
+++
```

Notes:
- Always set `draft = true` initially. The author publishes manually.
- Date format is ISO 8601 with timezone offset (Central Time: `-05:00` or
  `-06:00` depending on DST).
- Tags are lowercase. Existing tags in use: `aws`, `ai`, `homelab`, `kvm`,
  `libvirt`, `qemu`, `ubuntu`, `virtualization`, `vnc`, `ssh`, `cloud-init`.
- Categories in use: `certs`, `devops`, `homelab`.

---

## Writing Process

Follow this sequence when producing a blog post. Each phase requires the
author's input before proceeding to the next.

### Phase 1: Research and Context Gathering

Before writing anything:
1. Search past conversations using `conversation_search` and `recent_chats`
   to reconstruct the timeline and pull specific data points.
2. Search project knowledge if the post relates to project work.
3. Fetch the author's existing blog posts to match voice and style.
4. Ask clarifying questions about anything ambiguous. Do not assume.

### Phase 2: Structured Summary Document

Produce a summary document (Markdown) containing:
- Timeline of events with specific dates where known
- Key data points (scores, metrics, counts, versions)
- Honest assessment of what happened, what worked, what didn't
- Recommendations or lessons learned

Present this to the author for review before drafting the post. This
document is raw material, not the post itself.

### Phase 3: First Draft

Write the blog post using the voice and formatting rules below. The draft
should be a complete, publishable post, not an outline.

Present the draft for review. Expect multiple rounds of revision.

### Phase 4: Revision

Revise based on author feedback. Common revision requests:
- Tone adjustments (too formal, too casual, too AI-sounding)
- Adding or removing sections
- Inserting screenshots at specific points
- Correcting facts or timelines
- Restructuring for better narrative flow

### Phase 5: Screenshot Integration

When the author provides screenshots:
1. Determine placement within the post (which section, after which paragraph)
2. Write descriptive alt text
3. Use the correct image path convention: `/img/<post_slug>/filename.png`
4. Remind the author to create the directory and place files in
   `static/img/<post_slug>/`

---

## Writing Voice

The author's voice is conversational, honest, and self-aware. Match these
characteristics:

**Tone:**
- First person throughout
- Casual but not sloppy. Complete sentences, proper grammar, no txtspk.
- Direct. Say the thing without hedging or softening.
- Self-critical without being self-pitying. Acknowledge mistakes plainly.
- A bit of dry humor is fine, but don't force it.
- Occasional optimism, but earned, not performative.

**Paragraph structure:**
- Write in flowing paragraphs, not stacked single-line declarations.
- Each paragraph should contain multiple connected thoughts.
- Avoid the AI pattern of: short declarative sentence, period, another
  short declarative sentence, period. This reads as robotic.
- Thoughts within a paragraph should flow into each other naturally,
  the way someone would talk through an experience.

**What to avoid:**
- Marketing language ("game-changer", "deep dive", "unlock")
- Excessive praise or flattery of the author
- Filler phrases ("It should be noted that", "In order to")
- Overuse of bold for emphasis. Use it sparingly for key terms.
- Lists and bullet points in the body. Write in prose. Lists are acceptable
  only in recommendation/action-item sections where scannability matters.
- Em dashes. Use commas, periods, parentheses, or restructure the sentence
  instead. This is a hard rule with no exceptions.

**Opening pattern:**
- Posts open with "Welcome back friend." or a similar short, warm greeting.
- The first paragraph states what happened and sets the tone for the post.
- No preamble or throat-clearing. Get to the point immediately.

**Closing pattern:**
- Posts end with "Well, that's all. See you in the next one."
- A P.S. credits AI tools used during writing with the disclaimer that
  ideas and final edits are the author's own.

---

## Formatting Rules

- **Headings:** H1 for the post title (one only, at the top). H2 for major
  sections. H3 only if a section genuinely needs subdivision.
- **Links:** Use inline Markdown links. Link to other posts on the same site
  using relative paths when referencing them.
- **Code:** Use inline code for specific technical terms, commands, file paths,
  and tool names when they appear in prose. Use fenced code blocks only for
  multi-line code or commands the reader might copy.
- **Images:** Markdown syntax with descriptive alt text.
  `![Alt text description](/img/post_slug/filename.png)`
- **Emphasis:** Italics for titles of works, for contrast with surrounding
  text, or for words being used as words. Bold sparingly for key terms on
  first introduction only.
- **No em dashes.** Use commas, periods, parentheses, or sentence
  restructuring instead. Check the final output for any `—` characters
  before delivering.

---

## Content Principles

### Honesty Over Positivity

The author values honest self-assessment. If something went poorly, say so.
If a score is mediocre, don't spin it as a win. If effort was misallocated,
name the pattern specifically. Readers should trust that the post reflects
what actually happened, not a curated highlight reel.

### Specificity Over Generality

Name the tools, the scores, the dates, the files. "I scored 64% on my first
practice test" is better than "my initial scores were low." "I spent a
session debugging Mermaid dark mode rendering" is better than "I worked on
application improvements."

### Narrative Over List

The post should read as a story, not a report. Events should follow a
chronological or logical arc. The reader should understand not just what
happened but why, and what the author learned from it.

### Show the Work

When describing projects, processes, or decisions, include enough technical
detail that a fellow engineer would understand what was involved, but not
so much that it becomes a tutorial. The audience is technical but the format
is a blog post, not documentation.

---

## Pre-Delivery Checklist

Before presenting the final draft to the author:

1. [ ] Front matter is valid TOML with all required fields
2. [ ] `draft = true` is set
3. [ ] No em dashes (`—`) anywhere in the document
4. [ ] Images use `/img/<post_slug>/` path convention
5. [ ] Post opens with a greeting, closes with the standard sign-off
6. [ ] P.S. credits AI tools used
7. [ ] No bullet points in the narrative body (lists only in
       recommendation sections)
8. [ ] Paragraphs flow naturally (no stacked single-sentence declarations)
9. [ ] All facts, dates, and scores have been verified with the author or
       sourced from chat history/project knowledge
10. [ ] No unresolved assumptions. Anything uncertain was flagged as a
        question to the author before writing.

---

## Reference: Existing Posts

When matching voice and style, fetch and reference these published posts:
- `https://mrmcmuffinz.dev/posts/feburary_2026_update/`
- `https://mrmcmuffinz.dev/posts/i_passed_my_aws_ccp_cert/`

These establish the baseline for tone, paragraph structure, and formatting
conventions. New posts should feel like they were written by the same person.