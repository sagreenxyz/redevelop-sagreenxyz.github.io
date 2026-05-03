# Knowledge Base — AI Instructions

This file is the single source of truth for every AI assistant working on
this project. It covers the full stack, architecture, implementation
patterns, and content authoring workflow. Follow every section that applies
to the task at hand.

---

## Stack

| Layer | Choice |
|---|---|
| Framework | Astro (latest stable) |
| Styling | Tailwind CSS + DaisyUI |
| Content | Astro Content Collections + MDX |
| Interactivity | React islands (`@astrojs/react`) |
| Search | Pagefind (`astro-pagefind`) |
| Syntax highlighting | Shiki (Astro built-in) |
| Concordance engine | Custom build-time TypeScript in `src/lib/concordance/` |

Bootstrap command:
```bash
npm create astro@latest -- --template minimal
npx astro add tailwind mdx react
npm i daisyui astro-pagefind
```

`tailwind.config.mjs`:
```js
export default {
  content: ['./src/**/*.{astro,html,js,jsx,ts,tsx,mdx}'],
  plugins: [require('daisyui')],
  daisyui: {
    themes: ['light', 'dark', 'nord', 'dracula'],
    defaultTheme: 'light',
    darkTheme: 'dark',
  },
}
```

---

## Project Structure

```
src/
├── content/
│   ├── config.ts              ← Collection schemas — source of truth
│   ├── wiki/                  ← MDX articles, nestable
│   ├── docs/                  ← MDX structured guides
│   ├── blog/                  ← MDX blog posts
│   ├── changelog/             ← MDX release notes
│   └── tags/                  ← MDX glossary definitions, one file per term
│
├── layouts/
│   ├── BaseLayout.astro
│   ├── WikiLayout.astro
│   ├── DocsLayout.astro
│   ├── BlogLayout.astro
│   └── LandingLayout.astro
│
├── pages/
│   ├── index.astro
│   ├── search.astro
│   ├── wiki/
│   │   ├── index.astro
│   │   ├── tags/
│   │   │   ├── index.astro    ← A–Z glossary
│   │   │   └── [tag].astro   ← Single tag / concordance entry
│   │   └── [...slug].astro
│   ├── docs/
│   │   ├── index.astro
│   │   └── [...slug].astro
│   ├── blog/
│   │   ├── index.astro
│   │   └── [slug].astro
│   └── changelog/
│       └── index.astro
│
├── components/
│   ├── global/
│   │   ├── Navbar.astro
│   │   ├── Footer.astro
│   │   └── ThemeToggle.astro
│   ├── wiki/
│   │   ├── WikiSidebar.astro
│   │   ├── WikiBreadcrumb.astro
│   │   ├── WikiTOC.astro
│   │   ├── WikiCard.astro
│   │   ├── TagStrip.astro     ← Auto-rendered on every article
│   │   ├── BacklinksPanel.astro
│   │   └── TagTooltip.tsx     ← React island for hover definitions
│   └── ui/
│       ├── Callout.astro
│       └── CodeBlock.astro
│
└── lib/
    └── concordance/
        ├── types.ts
        ├── extract.ts         ← Orchestrates all extraction
        ├── tfidf.ts           ← Statistical term scoring
        ├── normalize.ts       ← Dedup, alias resolution
        └── cache.ts           ← SHA-256 hash-based build cache

.tag-cache/                    ← Build artifact, never commit
  manifest.json
```

`.gitignore` must include:
```
.tag-cache/
```

---

## Content Collections (`src/content/config.ts`)

```ts
import { defineCollection, z } from 'astro:content';

const articleBase = z.object({
  title: z.string(),
  description: z.string().optional(),
  draft: z.boolean().default(false),
  updatedAt: z.coerce.date().optional(),
  tags: z.array(z.string()).default([]),
});

const wiki = defineCollection({
  type: 'content',
  schema: articleBase.extend({
    category: z.string(),
    related: z.array(z.string()).default([]),
  }),
});

const docs = defineCollection({
  type: 'content',
  schema: articleBase.extend({
    section: z.string(),
    order: z.number().default(0),
  }),
});

const blog = defineCollection({
  type: 'content',
  schema: articleBase.extend({
    pubDate: z.coerce.date(),
    author: z.string().optional(),
  }),
});

const changelog = defineCollection({
  type: 'content',
  schema: z.object({
    version: z.string(),
    date: z.coerce.date(),
    breaking: z.boolean().default(false),
    summary: z.string(),
  }),
});

const tags = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string(),
    slug: z.string(),
    definition: z.string(),
    aliases: z.array(z.string()).default([]),
    related: z.array(z.string()).default([]),
    category: z.string().optional(),
    seeAlso: z.array(z.object({
      label: z.string(),
      url: z.string(),
    })).default([]),
    stub: z.boolean().default(false),
  }),
});

export const collections = { wiki, docs, blog, changelog, tags };
```

---

## Concordance System

### Core Principle

Tags on every article are **derived from content at build time**, not written
by hand. The concordance engine:

1. Runs TF-IDF across the full corpus to score terms by significance
2. Merges with any frontmatter tags the author explicitly wrote
3. Resolves aliases to canonical forms (`cached` → `caching`)
4. Writes a manifest that all page routes consume at render time

Authors never need to write tags. If they do, those tags are treated as
highest-confidence and merged in.

---

### Types (`src/lib/concordance/types.ts`)

```ts
export type ExtractedTag = {
  term: string;
  score: number;
  source: 'tfidf' | 'frontmatter';
  mentions: number;
  excerpts: string[];
};

export type ConcordanceEntry = {
  tag: string;
  definition: string;
  category?: string;
  stub: boolean;
  articles: {
    slug: string;
    title: string;
    collection: 'wiki' | 'docs' | 'blog';
    score: number;
    mentions: number;
    excerpts: string[];
    inFrontmatter: boolean;
  }[];
  relatedTags: string[];
};

export type Manifest = {
  byArticle: Record<string, ExtractedTag[]>;
  byTag: Record<string, ConcordanceEntry>;
  builtAt: string;
};
```

---

### TF-IDF (`src/lib/concordance/tfidf.ts`)

```ts
import type { ExtractedTag } from './types';

const STOPWORDS = new Set([
  'this','that','with','from','have','will','what','when','where',
  'which','their','there','been','were','they','them','then','than',
  'also','into','over','some','such','more','most','very','just',
  'each','much','both','only','does','after','before','about',
]);

export function computeTFIDF(
  docBody: string,
  corpus: Map<string, string>,
  topN = 20,
): Omit<ExtractedTag, 'excerpts'>[] {
  const docTokens = tokenize(docBody);
  const tf = calcTF(docTokens);
  const scores: { term: string; score: number; mentions: number }[] = [];

  for (const [term, freq] of tf.entries()) {
    const docsWithTerm = [...corpus.values()]
      .filter(b => b.toLowerCase().includes(term)).length;
    const idf = Math.log((corpus.size + 1) / (docsWithTerm + 1));
    scores.push({ term, score: freq * idf, mentions: Math.round(freq * docTokens.length) });
  }

  return scores
    .sort((a, b) => b.score - a.score)
    .slice(0, topN)
    .map(s => ({ ...s, source: 'tfidf' as const }));
}

function tokenize(text: string): string[] {
  return text
    .toLowerCase()
    .replace(/```[\s\S]*?```/g, ' ')
    .replace(/[^a-z0-9\s-]/g, ' ')
    .split(/\s+/)
    .filter(t => t.length > 3 && !STOPWORDS.has(t));
}

function calcTF(tokens: string[]): Map<string, number> {
  const counts = new Map<string, number>();
  for (const t of tokens) counts.set(t, (counts.get(t) ?? 0) + 1);
  const total = tokens.length;
  return new Map([...counts.entries()].map(([t, c]) => [t, c / total]));
}
```

---

### Normalizer (`src/lib/concordance/normalize.ts`)

```ts
import type { ExtractedTag } from './types';

export function normalize(
  tags: ExtractedTag[],
  aliasMap: Map<string, string>,
  minScore = 0.15,
  maxTags = 25,
): ExtractedTag[] {
  const merged = new Map<string, ExtractedTag>();

  for (const tag of tags) {
    const canonical = aliasMap.get(tag.term) ?? tag.term;
    const existing = merged.get(canonical);
    if (existing) {
      merged.set(canonical, {
        ...existing,
        term: canonical,
        score: Math.max(existing.score, tag.score),
        mentions: existing.mentions + tag.mentions,
        excerpts: [...new Set([...existing.excerpts, ...tag.excerpts])].slice(0, 3),
        source: tag.source === 'frontmatter' ? 'frontmatter' : existing.source,
      });
    } else {
      merged.set(canonical, { ...tag, term: canonical });
    }
  }

  return [...merged.values()]
    .filter(t => t.score > minScore)
    .sort((a, b) => {
      if (a.source === 'frontmatter' && b.source !== 'frontmatter') return -1;
      if (b.source === 'frontmatter' && a.source !== 'frontmatter') return 1;
      return b.score - a.score;
    })
    .slice(0, maxTags);
}

export function getExcerpts(body: string, term: string, windowSize = 120): string[] {
  const results: string[] = [];
  const lower = body.toLowerCase();
  let idx = lower.indexOf(term.toLowerCase());
  while (idx !== -1 && results.length < 3) {
    const start = Math.max(0, idx - windowSize / 2);
    const end = Math.min(body.length, idx + term.length + windowSize / 2);
    results.push('…' + body.slice(start, end).trim() + '…');
    idx = lower.indexOf(term.toLowerCase(), idx + 1);
  }
  return results;
}
```

---

### Cache (`src/lib/concordance/cache.ts`)

```ts
import { createHash } from 'crypto';
import { readFile, writeFile, mkdir } from 'fs/promises';
import type { Manifest } from './types';

const CACHE_DIR = '.tag-cache';
const MANIFEST_PATH = `${CACHE_DIR}/manifest.json`;

export function hashContent(body: string): string {
  return createHash('sha256').update(body).digest('hex').slice(0, 16);
}

export async function loadManifest(): Promise<Manifest | null> {
  try { return JSON.parse(await readFile(MANIFEST_PATH, 'utf8')); }
  catch { return null; }
}

export async function saveManifest(manifest: Manifest): Promise<void> {
  await mkdir(CACHE_DIR, { recursive: true });
  await writeFile(MANIFEST_PATH, JSON.stringify(manifest, null, 2));
}
```

---

### Main Extractor (`src/lib/concordance/extract.ts`)

```ts
import { getCollection } from 'astro:content';
import { computeTFIDF } from './tfidf';
import { normalize, getExcerpts } from './normalize';
import { saveManifest, loadManifest } from './cache';
import type { ExtractedTag, ConcordanceEntry, Manifest } from './types';

export async function buildConcordance(): Promise<Manifest> {
  const [wikiEntries, docsEntries, blogEntries, tagDefs] = await Promise.all([
    getCollection('wiki', e => !e.data.draft),
    getCollection('docs', e => !e.data.draft),
    getCollection('blog', e => !e.data.draft),
    getCollection('tags'),
  ]);

  const allContent = [
    ...wikiEntries.map(e => ({ ...e, collection: 'wiki' as const })),
    ...docsEntries.map(e => ({ ...e, collection: 'docs' as const })),
    ...blogEntries.map(e => ({ ...e, collection: 'blog' as const })),
  ];

  const aliasMap = new Map<string, string>();
  for (const tag of tagDefs) {
    aliasMap.set(tag.data.slug, tag.data.slug);
    for (const alias of tag.data.aliases) {
      aliasMap.set(alias.toLowerCase(), tag.data.slug);
    }
  }

  const corpus = new Map(allContent.map(e => [e.slug, e.body]));
  const byArticle: Manifest['byArticle'] = {};

  for (const entry of allContent) {
    const tfidfRaw = computeTFIDF(entry.body, corpus);
    const tfidfTags: ExtractedTag[] = tfidfRaw.map(t => ({
      ...t,
      excerpts: getExcerpts(entry.body, t.term),
    }));

    const frontmatterTags: ExtractedTag[] = (entry.data.tags ?? []).map(t => ({
      term: t.toLowerCase(),
      score: 1.0,
      source: 'frontmatter' as const,
      mentions: (entry.body.toLowerCase().match(new RegExp(`\\b${t}\\b`, 'g')) ?? []).length,
      excerpts: getExcerpts(entry.body, t),
    }));

    byArticle[entry.slug] = normalize([...frontmatterTags, ...tfidfTags], aliasMap);
  }

  const byTag: Manifest['byTag'] = {};

  for (const tag of tagDefs) {
    byTag[tag.data.slug] = {
      tag: tag.data.slug,
      definition: tag.data.definition,
      category: tag.data.category,
      stub: tag.data.stub,
      articles: [],
      relatedTags: tag.data.related,
    };
  }

  for (const entry of allContent) {
    for (const extracted of byArticle[entry.slug] ?? []) {
      if (!byTag[extracted.term]) {
        byTag[extracted.term] = {
          tag: extracted.term,
          definition: '',
          stub: true,
          articles: [],
          relatedTags: [],
        };
      }
      byTag[extracted.term].articles.push({
        slug: entry.slug,
        title: entry.data.title,
        collection: entry.collection,
        score: extracted.score,
        mentions: extracted.mentions,
        excerpts: extracted.excerpts,
        inFrontmatter: extracted.source === 'frontmatter',
      });
    }
  }

  for (const entry of Object.values(byTag)) {
    entry.articles.sort((a, b) => b.score - a.score);
  }

  const manifest: Manifest = { byArticle, byTag, builtAt: new Date().toISOString() };
  await saveManifest(manifest);
  return manifest;
}

export async function getTagsForSlug(slug: string): Promise<ExtractedTag[]> {
  const manifest = await buildConcordance();
  return manifest.byArticle[slug] ?? [];
}

export async function getConcordanceEntry(tag: string): Promise<ConcordanceEntry | null> {
  const manifest = await buildConcordance();
  return manifest.byTag[tag] ?? null;
}

export async function getAllTags(): Promise<ConcordanceEntry[]> {
  const manifest = await buildConcordance();
  return Object.values(manifest.byTag).sort((a, b) => a.tag.localeCompare(b.tag));
}
```

---

## Astro Config (`astro.config.mjs`)

```js
import { defineConfig } from 'astro/config';
import tailwind from '@astrojs/tailwind';
import mdx from '@astrojs/mdx';
import react from '@astrojs/react';
import pagefind from 'astro-pagefind';
import { loadManifest } from './src/lib/concordance/cache';

export default defineConfig({
  integrations: [tailwind(), mdx(), react(), pagefind()],
  markdown: {
    shikiConfig: { theme: 'dracula' },
  },
  hooks: {
    'astro:build:done': async () => {
      const manifest = await loadManifest();
      if (!manifest) return;
      const stubs = Object.values(manifest.byTag).filter(e => e.stub);
      if (stubs.length) {
        console.warn(`\n⚠️  ${stubs.length} tags have no definition file:`);
        stubs
          .sort((a, b) => b.articles.length - a.articles.length)
          .forEach(s => console.warn(`   ${s.tag}  (${s.articles.length} articles)`));
        console.warn(`   → create src/content/tags/<slug>.mdx for each\n`);
      }
    },
  },
});
```

---

## Layouts

All imports use the `@/` alias (maps to `src/`).

### `BaseLayout.astro`
- Sets `<html data-theme="light">`
- Includes `<Navbar>` and `<Footer>`
- Accepts `title` and `description` props for `<head>`
- `<ThemeToggle>` swaps `data-theme` between light and dark

### `WikiLayout.astro`
Uses DaisyUI `drawer` for sidebar layout:
```astro
<BaseLayout {title}>
  <div class="drawer lg:drawer-open">
    <input id="drawer-toggle" type="checkbox" class="drawer-toggle" />
    <div class="drawer-content flex flex-col">
      <WikiBreadcrumb {entry} />
      <article class="prose max-w-none p-6" data-pagefind-body>
        <h1>{entry.data.title}</h1>
        <TagStrip slug={entry.slug} />
        <slot />
      </article>
      <BacklinksPanel {backlinks} />
    </div>
    <div class="drawer-side">
      <WikiSidebar {tree} />
    </div>
  </div>
</BaseLayout>
```

### `DocsLayout.astro`
Same drawer pattern as WikiLayout plus a right-hand sticky `WikiTOC` column.

### `BlogLayout.astro`
Centered single column, `prose-lg`. Author, date, and `TagStrip` in the header.

### `LandingLayout.astro`
Full-width, no sidebar. Used for the homepage and index pages.

---

## Components

### `TagStrip.astro`
Reads `getTagsForSlug(slug)` and renders DaisyUI badges.
- `score >= 0.6` → `badge badge-primary`
- `score < 0.6` → `badge badge-ghost text-xs`
- Every badge links to `/wiki/tags/[term]`

### `TagTooltip.tsx` (React island, `client:load`)
```tsx
export default function TagTooltip({
  tag, definition, children
}: { tag: string; definition: string; children: React.ReactNode }) {
  return (
    <span className="tooltip tooltip-bottom cursor-help" data-tip={definition}>
      <a href={`/wiki/tags/${tag}`} className="underline decoration-dotted">
        {children}
      </a>
    </span>
  );
}
```

### `WikiSidebar.astro`
DaisyUI `menu` with nested `<ul>` for categories. Active page has `menu-active`.
Add `data-pagefind-ignore` to this element.

### `WikiTOC.astro`
Accepts `headings` from `entry.render()`. Renders a vertical DaisyUI `menu`
of anchor links. Sticky-positioned in DocsLayout's right column.

### `WikiBreadcrumb.astro`
DaisyUI `breadcrumbs`. Path derived from splitting the entry slug on `/`.

### `BacklinksPanel.astro`
DaisyUI `collapse collapse-arrow` at the bottom of every wiki article,
listing other articles that share tags with this one.

### `Callout.astro`
Wraps DaisyUI `alert` with a `type` prop: `info`, `warning`, `error`, `success`.

---

## Page Routes

### `/wiki/[...slug].astro`
```astro
---
import { getCollection } from 'astro:content';
import WikiLayout from '@/layouts/WikiLayout.astro';
import { buildConcordance } from '@/lib/concordance/extract';

export async function getStaticPaths() {
  const entries = await getCollection('wiki', e => !e.data.draft);
  return entries.map(e => ({ params: { slug: e.slug }, props: { entry: e } }));
}
const { entry } = Astro.props;
const { Content, headings } = await entry.render();
---
<WikiLayout {entry} {headings}>
  <Content />
</WikiLayout>
```

### `/wiki/tags/index.astro`
- Fetch all tags via `getAllTags()`
- Group by first character
- DaisyUI `tabs tabs-bordered` for alphabet navigation
- Each entry: name (linked), stub `badge-warning` if applicable, article count, definition

### `/wiki/tags/[tag].astro`
```astro
export async function getStaticPaths() {
  const tags = await getAllTags();
  return tags.map(t => ({ params: { tag: t.tag } }));
}
```
Page sections in order:
1. **Definition** — `alert alert-info` with one-sentence definition
2. **Full MDX body** — if a definition file exists, render its `<Content>` in `<article class="prose">`
3. **Related tags** — `badge-outline` links
4. **Concordance table** — `table table-zebra`, columns: Article, Collection, Mentions, Excerpt; sorted by mentions descending
5. **See also** — external links from `seeAlso`

### `/search.astro`
```astro
<BaseLayout title="Search">
  <div class="max-w-3xl mx-auto px-4 py-8">
    <h1 class="text-3xl font-bold mb-6">Search</h1>
    <div id="search"></div>
  </div>
</BaseLayout>
<link href="/pagefind/pagefind-ui.css" rel="stylesheet" />
<script src="/pagefind/pagefind-ui.js"></script>
<script>new PagefindUI({ element: '#search', showSubResults: true });</script>
```

---

## DaisyUI Component Map

| UI need | DaisyUI component |
|---|---|
| Wiki page shell | `drawer`, `drawer-open` |
| Sidebar navigation | `menu`, nested `ul` |
| Breadcrumbs | `breadcrumbs` |
| Tag badges | `badge`, `badge-primary`, `badge-ghost`, `badge-outline` |
| Callout blocks | `alert`, `alert-info`, `alert-warning`, `alert-error` |
| Concordance table | `table`, `table-zebra` |
| Search modal | `modal`, `modal-box` |
| A–Z glossary nav | `tabs`, `tabs-bordered` |
| Article preview cards | `card`, `card-body` |
| Collapse / backlinks | `collapse`, `collapse-arrow` |
| Theme toggle | `swap`, `swap-rotate` |
| Inline tooltips | `tooltip`, `tooltip-bottom` |
| Table of contents | `menu` (vertical, sticky) |

---

## Code Conventions

- All imports use the `@/` alias mapped to `src/`
- File names: lowercase kebab-case everywhere
- Tag slugs: lowercase kebab-case, must match filename and `slug` frontmatter field
- React islands are `.tsx` and always carry a `client:` directive at the call site
- Never put interactive logic in `.astro` files — delegate to `.tsx` islands
- `buildConcordance()` is build-time only — never import it in client-side code
- Draft articles (`draft: true`) are excluded from `getStaticPaths` and from the concordance corpus
- Add `data-pagefind-body` to article `<article>` elements
- Add `data-pagefind-ignore` to `<Navbar>`, `<Footer>`, and `<WikiSidebar>`

---

## Content Authoring

This section explains how content is created and maintained. When the user
asks for help writing or organizing content, follow these rules exactly.

### The One Rule

Tags are automatic. The author writes content; the system derives tags at
build time via TF-IDF. Authors never need to write or maintain tags manually.
If an author adds `tags: [something]` to frontmatter, those are treated as
highest-confidence and merged in — but it is never required.

### File Locations and URL Mapping

```
src/content/wiki/caching.mdx                 → /wiki/caching
src/content/wiki/systems/caching.mdx         → /wiki/systems/caching
src/content/wiki/systems/http/caching.mdx    → /wiki/systems/http/caching
src/content/docs/getting-started.mdx         → /docs/getting-started
src/content/blog/my-post.mdx                 → /blog/my-post
src/content/tags/caching.mdx                 → /wiki/tags/caching
```

Nesting is unlimited. The folder path becomes the URL slug automatically.

### Wiki Article Frontmatter

```yaml
---
title: Cache Invalidation
description: Strategies for expiring stale cached data when the source changes.
category: Systems
related: []
draft: false
---
```

`title` — Required.
`description` — One or two sentences. Used in search results and cards.
`category` — Groups articles in the sidebar. Use consistent names: `Systems`, `Frontend`, `Security`, `Databases`.
`related` — Other wiki article slugs. Optional.
`draft` — Set `true` to hide from the built site.
`tags` — Omit entirely. The concordance fills this. Only add if forcing a specific tag.
`updatedAt` — Set on significant revisions. Format: `YYYY-MM-DD`.

### Docs Article Frontmatter

```yaml
---
title: Getting Started
description: Install and configure the system in five minutes.
section: Introduction
order: 10
draft: false
---
```

`section` — Groups docs pages in the left navigation.
`order` — Sort order within a section. Use increments of 10.

### Blog Post Frontmatter

```yaml
---
title: Why We Rebuilt the Cache Layer
description: A walkthrough of the decisions behind our new caching strategy.
pubDate: 2025-06-15
author: Author Name
draft: false
---
```

### Tag Definition Frontmatter

```yaml
---
title: Cache Invalidation
slug: cache-invalidation
definition: "The process of removing or replacing stale cache entries when the source data changes."
aliases: [cache busting, cache expiry]
related: [caching, ttl]
category: Systems
stub: false
seeAlso:
  - label: "MDN: Cache-Control"
    url: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control
---
```

`slug` — Must match the filename exactly. Lowercase kebab-case.
`definition` — One sentence, under 20 words. Used in tooltips, glossary index, and tag pages.
`aliases` — Other forms that resolve to this term. `["cache busting"]` means articles mentioning "cache busting" link here.
`related` — Other tag slugs. Creates a "related terms" section.
`stub` — `true` means auto-detected, no human definition yet. Set to `false` once written.

The body of a tag file is optional. A `definition` field alone is sufficient.
Add a full body only when the term warrants a longer explanation.

### Components Available in MDX

**Callout boxes:**
```mdx
import Callout from '@/components/ui/Callout';

<Callout type="info">This feature requires version 2.0 or later.</Callout>
<Callout type="warning">Changing this setting restarts the process.</Callout>
<Callout type="error">Do not use in production without rate limiting.</Callout>
<Callout type="success">This is the current recommended pattern.</Callout>
```

**Inline term tooltip:**
```mdx
import TagTooltip from '@/components/wiki/TagTooltip';

We use <TagTooltip tag="caching" definition="Storing copies of data to reduce latency">caching</TagTooltip> at the edge.
```

Use `TagTooltip` only when hover behaviour is wanted inline in a sentence.
Tag badges at the bottom of the article are automatic and do not require this component.

### The Build Warning Loop

After every `npm run build`, the concordance engine prints terms that appear
in articles but have no definition file:

```
⚠️  6 tags have no definition file:
   cache-invalidation  (12 articles)
   rate-limiting       (8 articles)
   eventual-consistency (5 articles)
   → create src/content/tags/<slug>.mdx for each
```

The list is sorted by article count. Work top-to-bottom. A tag definition
needs only a `slug` and `definition` to clear the warning. A full MDX body
is optional.

### Content Workflow

**Adding a new topic:**
1. Create `src/content/wiki/<slug>.mdx` with frontmatter and body.
2. Run `npm run dev` to preview.
3. Run `npm run build` and read the warning output.
4. Create `src/content/tags/<slug>.mdx` for each flagged term.
5. Rebuild until warnings are clear.

**Expanding an existing topic:**
1. Edit the MDX body. The concordance updates on next build automatically.
2. Bump `updatedAt` in frontmatter if the change is substantial.
3. Run build and handle any new warnings.

**Starting a glossary from scratch:**
1. Write 5–10 articles without worrying about tags.
2. Run `npm run build`.
3. The warning list is your auto-generated starter glossary, ranked by importance.
4. Work through it top-to-bottom, writing a definition file for each term.

### What the Author Never Needs to Do

- Write or maintain tags on articles
- Update tags when editing an article
- Create backlinks between articles manually
- Build or maintain the glossary index page
- Update "related articles" sections
- Track which articles mention a given term

The system handles all of this automatically.