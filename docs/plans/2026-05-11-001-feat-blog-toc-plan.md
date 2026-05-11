---
title: "feat: Add responsive blog table of contents"
type: feat
status: active
date: 2026-05-11
---

# feat: Add responsive blog table of contents

## Overview

Add a polished post table-of-contents UI inspired by `https://ednutting.com/2025/07/29/claude-website-rewrite.html`.

This repo already renders Hugo's built-in `{{ .TableOfContents }}` in `layouts/_default/single.html`, so the work should reuse Hugo's generated TOC instead of building a new heading parser.

---

## Problem Frame

Blog posts currently show a plain inline Hugo TOC before the article body. The desired behavior is a more intentional reading/navigation affordance: a visible "Contents" block on desktop and a compact mobile-friendly "Table of Contents" disclosure, without hurting readability or adding unnecessary JavaScript.

---

## Requirements Trace

- R1. Reuse Hugo's generated `.TableOfContents` as the source of truth.
- R2. Present the TOC as a designed "Contents" component similar to the reference page.
- R3. Keep the article readable on desktop and mobile.
- R4. Avoid duplicate `#TableOfContents` IDs.
- R5. Preserve no-JS usability.
- R6. Support dark mode and keyboard navigation.
- R7. Avoid showing noisy empty/minimal TOCs when posts do not have meaningful headings.

---

## Scope Boundaries

- Do not build active-section scrollspy in v1.
- Do not parse headings manually in JavaScript.
- Do not add a new asset pipeline or dependency.
- Do not redesign the whole post layout beyond what the TOC needs.

---

## Context & Research

### Relevant Code and Patterns

- `layouts/_default/single.html` currently renders `{{ .TableOfContents }}` before `{{ .Content }}`.
- `hugo.toml` configures `[markup.tableOfContents]` with `startLevel = 1` and `endLevel = 3`.
- `static/css/main.css` already styles `#TableOfContents` and uses `prefers-color-scheme` for dark mode.
- `.github/workflows/hugo.yaml` builds with Hugo `0.125.4`; `.tool-versions` uses Hugo `0.155.3`, so implementation should avoid newer Hugo-only template features unless verified against CI.
- `justfile` exposes `just serve`, which runs `hugo serve`.

### Institutional Learnings

- No relevant `docs/solutions/` learning docs exist.

### External References

- Hugo `.TableOfContents` returns a `<nav id="TableOfContents">` containing list links to Markdown headings.
- Hugo's current docs confirm the default TOC heading range is configurable via `[markup.tableOfContents]`.

---

## Key Technical Decisions

- Reuse one Hugo TOC render: Avoid duplicate IDs and keep heading behavior consistent with Hugo.
- Prefer native `<details>/<summary>` for mobile disclosure: This gives keyboard and no-JS behavior without adding a script.
- Style with CSS only in `static/css/main.css`: The repo has plain static CSS and no local JS files.
- Desktop TOC as sidebar rail: On wide screens, position the TOC as a sticky sidebar rail in the space outside the existing `max-width: 720px` content column, using the available margin rather than shrinking the readable area. This requires extending the layout beyond the current centered single-column approach.
- Hide empty or non-meaningful TOCs where feasible: A styled card makes low-value TOCs more noticeable, so templates should avoid rendering one when it adds clutter.

---

## Open Questions

### Resolved During Planning

- Should this use Hugo or custom JS heading extraction? Use Hugo `.TableOfContents`.
- Should mobile require JavaScript? No; use native disclosure behavior.
- Should the implementation duplicate desktop/mobile TOCs? No; render once.

### Deferred to Implementation

- Exact Hugo conditional for hiding one-heading TOCs: choose the simplest CI-compatible template check after validating generated output.
- Exact desktop breakpoint and sticky offset: tune while visually checking existing post pages.

---

## High-Level Technical Design

> This illustrates the intended approach and is directional guidance for review, not implementation specification. The implementing agent should treat it as context, not code to reproduce.

```text
single post layout
  post header
  toc wrapper
    details/summary label for mobile affordance
    Hugo .TableOfContents rendered once
  article content
  tags / related / comments

CSS
  small screens: compact disclosure, scrollable expanded panel
  large screens: sidebar rail in left/right margin, sticky positioning
  dark mode: card, border, links, focus states
```

---

## Implementation Units

- [ ] U1. **Wrap the existing Hugo TOC semantically**

**Goal:** Convert the current bare `{{ .TableOfContents }}` render into a reusable post TOC component shape while rendering Hugo's TOC exactly once.

**Requirements:** R1, R2, R4, R5, R7

**Dependencies:** None

**Files:**
- Modify: `layouts/_default/single.html`
- Test: `content/posts/online_python_memory_profiling.md`
- Test: `content/posts/trip-to-jp.md`

**Approach:**
- Replace the bare TOC with a wrapper such as a post TOC section near the post header.
- Use native disclosure markup for mobile-friendly collapsed behavior.
- Style the `<summary>` element with appropriate font weight and padding. Keep the browser's default disclosure triangle marker; do not replace it with a custom icon. Expansion should be instant (no animation) to avoid CSS complexity.
- Keep Hugo's generated `<nav id="TableOfContents">` as the only TOC nav in the DOM.
- Add accessible labeling text like "Table of Contents" / "Contents".
- Add a conditional so posts without useful headings do not render an empty card.

**Patterns to follow:**
- Existing post layout in `layouts/_default/single.html`.
- Hugo `.TableOfContents` behavior from `hugo.toml`.

**Test scenarios:**
- Happy path: A post with multiple `#`, `##`, and `###` headings renders a TOC with links before article content.
- Edge case: A post with no useful headings does not show an empty "Contents" card.
- Edge case: The rendered HTML contains only one `id="TableOfContents"`.
- Integration: Clicking a TOC link navigates to the matching heading anchor generated by Hugo.

**Verification:**
- Built post pages retain content, metadata, tags, related posts, comments, and TOC links.
- The generated TOC source is still Hugo-managed.

---

- [ ] U2. **Style the TOC for desktop, mobile, and dark mode**

**Goal:** Make the TOC look and behave like an intentional reading aid inspired by the reference page.

**Requirements:** R2, R3, R5, R6

**Dependencies:** U1

**Files:**
- Modify: `static/css/main.css`
- Test: `content/posts/online_python_memory_profiling.md`
- Test: `content/posts/trip-to-jp.md`

**Approach:**
- Replace or extend the current `#TableOfContents` rules under the existing TOC section.
- On mobile, show a compact "Table of Contents" disclosure that does not consume the first screen by default.
- On desktop, show a "Contents" card with comfortable spacing, nested heading indentation, and a bounded max height for long TOCs.
- Add `overflow-y: auto` for long TOCs.
- Add dark-mode background, border, text, hover, and focus styles.
- Add visible `:focus` styles for TOC links in both light and dark modes (e.g., outline or underline) since the existing CSS only has `:hover` with no `:focus` rule.
- Fix the existing dark-mode gap: `#TableOfContents a` is `color: #555` with no `prefers-color-scheme: dark` override, so TOC links are unreadable on dark backgrounds until an explicit dark-mode link color is added.
- Add heading `scroll-margin-top` if sticky/fixed UI can overlap anchors.

**Patterns to follow:**
- Existing responsive CSS and `prefers-color-scheme` patterns in `static/css/main.css`.
- Existing typography choices: `Lora` body text and `Open Sans` headings/UI.

**Test scenarios:**
- Happy path: On desktop width, TOC appears as a styled "Contents" card and article column remains readable.
- Happy path: On mobile width, TOC appears as a compact disclosure and expands to show links.
- Edge case: Long TOCs scroll internally rather than exceeding the viewport.
- Edge case: Nested `h2`/`h3` links remain visually distinguishable.
- Edge case: Dark mode preserves readable contrast for card text, links, borders, and hover/focus states.
- Integration: Heading anchor jumps land with headings visible, not hidden under sticky UI.

**Verification:**
- Existing post styling outside the TOC is unchanged.
- TOC is usable with keyboard and touch.
- No new CSS dependency or build step is introduced.

---

- [ ] U3. **Tune Hugo TOC heading depth if needed**

**Goal:** Ensure the TOC contains useful article sections without duplicating the post title or becoming too noisy.

**Requirements:** R1, R3, R7

**Dependencies:** U1, U2

**Files:**
- Modify: `hugo.toml`
- Test: `content/posts/online_python_memory_profiling.md`
- Test: `content/posts/drive_italy.md`

**Approach:**
- Review whether `startLevel = 1` causes redundant top-level entries for posts whose content starts with `#`.
- Prefer `startLevel = 2` if the post title and first content heading make `h1` entries noisy.
- Keep `endLevel = 3` unless visual testing shows `h3` creates too much clutter.

**Patterns to follow:**
- Existing `[markup.tableOfContents]` config in `hugo.toml`.
- Hugo's documented defaults for table of contents configuration.

**Test scenarios:**
- Happy path: A technical post with nested sections shows meaningful section links.
- Edge case: A post with a first `#` heading does not create a confusing duplicate-title experience.
- Edge case: A post with only shallow headings still has a useful TOC or no TOC if not meaningful.

**Verification:**
- Generated TOCs reflect the intended heading levels across representative posts.

---

- [ ] U4. **Verify build and representative pages**

**Goal:** Confirm the plan works across the static site, not just one reference post.

**Requirements:** R1, R2, R3, R5, R6, R7

**Dependencies:** U1, U2, U3

**Files:**
- Test: `.github/workflows/hugo.yaml`
- Test: `justfile`
- Test: `content/posts/online_python_memory_profiling.md`
- Test: `content/posts/trip-to-jp.md`
- Test: `content/posts/drive_italy.md`

**Approach:**
- Build with the repo's Hugo setup.
- Inspect generated pages for one technical long post, one travel/photo-heavy post, and one shorter post.
- Confirm no JS errors are introduced, because the implementation should not require JS.
- Confirm the GitHub Pages build path remains compatible with `baseurl` and `canonifyURLs`.

**Patterns to follow:**
- Existing CI build in `.github/workflows/hugo.yaml`.
- Existing local workflow in `justfile`.

**Test scenarios:**
- Integration: Static build completes with no Hugo template errors.
- Integration: A long post renders a usable TOC.
- Integration: A short/no-heading post does not render a broken or empty TOC.
- Integration: Site assets still load under the `/roaming` base URL.
- Error path: If Hugo conditional logic is incompatible with CI's Hugo `0.125.4`, simplify the conditional rather than relying on newer Hugo features.

**Verification:**
- Local generated site matches intended layout on desktop and mobile.
- CI-compatible Hugo build remains valid.

---

## System-Wide Impact

- **Interaction graph:** Affects only single post rendering through `layouts/_default/single.html` and global CSS in `static/css/main.css`.
- **Error propagation:** Hugo template mistakes can fail the static build; keep template logic simple and CI-compatible.
- **State lifecycle risks:** No persistent state or runtime data.
- **API surface parity:** No external API changes.
- **Integration coverage:** Build and visual checks are the meaningful integration coverage.
- **Unchanged invariants:** Post content, tags, related posts, comments, Giscus, syntax highlighting, KaTeX, and SEO partials should remain unchanged.

---

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| Duplicate `#TableOfContents` IDs from separate desktop/mobile renders | Render Hugo `.TableOfContents` once and adapt with CSS |
| Mobile TOC consumes too much vertical space | Use native disclosure collapsed by default |
| Long TOCs exceed viewport | Add max height and internal scrolling |
| Dark mode becomes low contrast | Add explicit dark-mode card/link styles |
| Hugo version mismatch between local and CI | Avoid newer template features or verify against CI Hugo `0.125.4` |
| Empty TOC card appears on short posts | Add conservative conditional rendering or hide empty TOC with CSS fallback |

---

## Documentation / Operational Notes

- No user-facing docs are required.
- If implementation uncovers Hugo template compatibility details, capture them later in `docs/solutions/`.

---

## Sources & References

- Related code: `layouts/_default/single.html`
- Related code: `static/css/main.css`
- Related config: `hugo.toml`
- Related workflow: `.github/workflows/hugo.yaml`
- Reference page: `https://ednutting.com/2025/07/29/claude-website-rewrite.html`
- Hugo docs: `https://gohugo.io/methods/page/tableofcontents/`
- Hugo docs: `https://gohugo.io/configuration/markup/#table-of-contents`
