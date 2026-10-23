---
name: seo-audit
description: >
  Technical + on-page SEO audit for a website or HubSpot/WordPress theme. Crawls (or reads the
  local theme source for) a set of pages and produces a prioritized Issue List covering meta tags
  (title/description/canonical/OG/Twitter), structured data (JSON-LD validity + HTML-entity
  encoding), heading hierarchy, images (weight/format/alt quality), links (dead/hidden-row),
  accessibility of decorative vs meaningful media, and page weight. Separates fixes that belong in
  THEME CODE from fixes that belong in the CMS EDITOR / FILE MANAGER, because most content-level
  issues cannot be fixed in the repo. TRIGGER on: "seo audit", "/seo-audit", "audit SEO", "check
  SEO", "pre-launch SEO", "audit these pages for SEO", "technical SEO review", or a pasted list of
  URLs to audit. Distinct from /launch-audit (broad go-live QA); this one is SEO-specific and
  entity/schema-aware.
author: Roshin Jose
author_email: roshinjose1@gmail.com
author_phone: "+919645228131"
---

# SEO Audit

You produce a **prioritized, evidence-backed SEO Issue List** — never vague advice. Every finding
names the page, the exact evidence (tag text, byte size, heading sequence), a severity, and a
**fix owner**: `THEME CODE` (editable in the repo) or `CMS EDITOR` (page settings / File Manager /
dnd content that only exists in the live CMS). This split is the single most important thing this
skill gets right — on HubSpot/WordPress themes, image binaries, per-page alt text, meta
descriptions, and hidden dnd rows usually do **not** live in the theme repo, so telling the user
"I compressed the images" when they're File-Manager assets is wrong.

## 0. First move

1. **Establish the two inputs.** (a) The live site / list of URLs, and (b) the theme source dir if
   there is one. Ask for whichever is missing. You need both: the live crawl finds *symptoms*, the
   source tells you *whether the fix is code or content*.
2. **Fetch each page** (WebFetch for live URLs; Read for local templates/modules). For a HubSpot
   theme, map each audited URL to the template + modules that build it before claiming a fix is
   code-side.
3. **Confirm before destructive calls.** Deleting hidden CMS rows, changing canonicals, or editing
   live meta are the user's calls — surface them, don't silently do them.

## 1. What to check (the audit dimensions)

Run every dimension against every audited page. For each finding record:
`{ id, page, dimension, evidence, severity (Blocker/High/Med/Low), owner (THEME CODE|CMS EDITOR), fix }`

### A. Meta & head
- `<title>` present, unique, ≤ ~60 chars, not the raw page name.
- `meta description` present, unique, 120–160 chars, **factually consistent with structured data**
  (e.g. don't say "25 years" when `foundingDate` implies 31 — cross-check numbers against schema
  and footer).
- Canonical present and self-referential (or intentionally cross-domain).
- **Open Graph**: `og:title`, `og:description`, `og:type`, `og:url`, `og:image` (1200×630, <300KB).
  Missing `og:image` is High — kills social/link-preview CTR. Twitter: `summary_large_image` + image.
- Robots meta / noindex not accidentally set on live pages.

### B. Structured data (JSON-LD) — be entity-aware
- Validate each block parses as JSON. The classic HubL/Jinja bug: emitting `|escape` (HTML-escapes
  `&`→`&amp;`) inside JSON-LD, so `Energy & Power` becomes `Energy &amp; Power` in the entity name.
  JSON-LD needs the literal `&`; only `"` and `\` need escaping. Flag any `&amp;`/`&#38;`/`&lt;`
  inside a JSON-LD string value.
- **Empty collection schema**: `ItemList`/`BreadcrumbList` with `"itemListElement": []`. Either
  populate or suppress the whole `<script>` — never emit an empty list. Common cause: the wrapper
  `{% if items %}` is looser than the per-item filter inside the loop, so all items get filtered out
  but the script still renders.
- Trailing-comma / malformed JSON from `{% if not loop.last %}` sitting *inside* a per-item filter.
- Organization/LocalBusiness: `foundingDate`, `name`, `logo`, `sameAs` consistent across the site.

### C. Heading hierarchy
- Exactly one `<h1>` per page.
- No skipped levels (H1→H3, H2→H4). Two root causes to distinguish:
  - **Structural** (THEME CODE): a module hardcodes `<h4>` for what should be `<h3>`. Fix the tag;
    preserve styling by moving/duplicating the CSS selector (e.g. `.card h4`→`.card h3`) or adding a
    class so appearance doesn't change.
  - **Content** (CMS EDITOR): an intermediate heading is *empty on the live page* (editor cleared
    the section title), so an H2 collapses and the H3 below it looks like it skips. Fix = restore
    the heading text, not the code.

### D. Images
- Flag any image > ~200KB (thumbnail/card) or > ~400KB (hero). Give target <150–200KB, and
  recommend WebP/AVIF + correct intrinsic dimensions + `loading="lazy"` for below-fold.
- **Distinguish `<img>` from CSS `background-image`** — background images have no alt and are
  decorative; they're a *weight* finding only, never an *alt* finding.
- Alt quality (not just presence): raw filenames (`DSC00052.jpg`, `Group 21`, `asset-7`,
  design-tool/camera/stock IDs, upload artifacts like `1280px-Foo.svg`) are effectively missing alt.
  Rank these *below* truly-empty alts in intentionality but still fix.
- **Decorative duplicate sets** (logo marquees, carousels that clone slides for a seamless loop):
  the cloned set SHOULD have `alt=""` + `aria-hidden="true"`. Confirm that's intentional — it is the
  correct pattern, not a bug. Only the *primary* set needs meaningful alt.

### E. Links & hidden content
- Dead/`#`/404 links, especially in nav/footer (global → every page).
- **Hidden CMS rows still in the page build**: dnd rows toggled hidden but shipped in the DOM,
  often containing dead links. Recommend deleting the row outright rather than leaving it hidden.
  This is CMS EDITOR work.

### F. Page weight & misc
- Total page weight, render-blocking assets, missing `width`/`height` (CLS), missing favicon,
  sitemap/robots.txt presence, llms.txt (optional).

## 2. THEME CODE vs CMS EDITOR — the ownership table

Default ownership (verify against the actual source before asserting):

| Issue | Usually owned by |
|---|---|
| og/twitter tags, canonical scaffold | THEME CODE (base layout `<head>`) |
| JSON-LD template + entity encoding + empty-list guard | THEME CODE (module/template) |
| Heading *tag level* in a module | THEME CODE |
| Heading *empty on page* | CMS EDITOR |
| Image binary / compression / format | CMS EDITOR (File Manager) — unless the file is in the repo `images/` |
| Per-image alt text values | CMS EDITOR (File Manager / module field) — code can only set a *fallback* for empty alts |
| Meta description / title text | CMS EDITOR (page settings) |
| Hidden dnd rows, page-level content | CMS EDITOR |
| Alt *fallback logic* (`alt or company_name`) | THEME CODE |

**Code can add fallbacks** (`alt="{{ field.alt or item.title }}"`) that fix *empty* alts, but a
stored bad filename (`DSC00052.jpg`) is a real value and overrides the fallback — that must be fixed
in the editor.

## 3. Output — the Issue List

Produce a table sorted by severity, then a short "code changes applied" section and a "CMS editor
to-do" checklist the user can hand to whoever owns the live site. Use this shape:

```
## SEO Audit — <site> — <date>
### Summary: N findings (x Blocker, y High, …). A applied in code, B need CMS editor.

| ID | Page | Dimension | Severity | Evidence | Fix | Owner |
|----|------|-----------|----------|----------|-----|-------|

### Applied in theme code (ready to upload)
- …files changed…

### Needs CMS editor / File Manager (cannot be done from repo)
- [ ] …
```

For a visually rich hand-off, offer to render the report as an Artifact (load `artifact-design`
first). Keep the raw markdown table too — the user often pastes it into a ticket.

## 4. Self-check
- [ ] Every finding has page + evidence + severity + owner.
- [ ] No finding claims a content asset was "fixed in code" when it's a File-Manager/editor asset.
- [ ] JSON-LD findings distinguish "invalid JSON", "wrong entity encoding", and "empty list".
- [ ] Heading findings distinguish structural (code) vs empty-heading (content).
- [ ] Image findings distinguish `<img>` alt from `background-image` weight.
- [ ] Decorative duplicate media confirmed as intentional, not flagged as missing-alt.
- [ ] Destructive/live changes surfaced for confirmation, not silently performed.
