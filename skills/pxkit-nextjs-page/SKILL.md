---
name: pxkit-nextjs-page
description: Build a Next.js App Router page, route group, or landing section the pxkit way — a thin server page composing section components, client code only in leaves, metadata through the root title template and a buildPageMetadata function, routes registered in one typed config. Use whenever the user asks for a new page, route, landing or marketing section, hero, pricing or about page, layout, route group, SEO metadata, or sitemap entry, even when they only say "add a page for X".
---

# pxkit Next.js Page

Every Next.js routing and metadata rule lives here. Load `pxkit-conventions` for the rules that apply to all code; its `components.md` holds the server/client boundary rules.

## Gates

- **The page file only composes sections.** No business logic, no markup beyond layout.
- **No layout wrapper or section base component invented for one page.** Use the repo's container, otherwise a plain `<section>`.
- **No route added without registering it** in the repo's typed routes config, when it has one.

## 1. The page

A thin server component, `export default function` (Next.js requires the default export):

```tsx
// app/(marketing)/page.tsx
export const metadata = buildPageMetadata({
  title: 'Pricing',
  description: 'Plans for every team size.',
  path: '/pricing',
});

export default function PricingPage() {
  return (
    <>
      <section><PricingHero /></section>
      <section><PricingTable /></section>
    </>
  );
}
```

Wrap sections in the repo's layout or container component when one exists.

## 2. Sections

- Server components, colocated with the route; moved to shared `components/` only when a second route uses them.
- A section's markup is readable in its own file: content as a typed `as const` array in that file, data fetched by the section that renders it.
- Keep the tree page → section → primitive.
- Interactivity goes in a leaf: a section stays a server component and renders a small `'use client'` child.

## 3. Routing

- Route groups `(folder)` organize by domain without changing URLs (`(marketing)`, `(dashboard)`), each with its own `layout.tsx`.
- A route folder owns its feature code (see `pxkit-feature`). Only code shared across routes goes to top-level folders.
- Use the framework's file conventions (`loading.tsx`, `error.tsx`, `not-found.tsx`, `sitemap.ts`, `robots.ts`, route handlers) instead of hand-rolling them.
- Navigation and `sitemap.ts` both read one typed routes config; a path is written once.

## 4. Metadata

- The root layout sets `metadataBase: new URL(siteConfig.url)` (relative Open Graph and canonical URLs break without it) and `title: { template: '%s - Site Name', default: 'Site Name — tagline' }`. Pages set a bare `title`; the home page omits it and gets `default`.
- Every page has `description`, `openGraph`, `twitter`, and `alternates.canonical`.
- Once three pages repeat the same metadata shape, build it with one plain function:

```ts
// lib/page-metadata.ts
interface PageMetadataInput {
  title: string;
  description: string;
  path: string;
}

export const buildPageMetadata = ({ title, description, path }: PageMetadataInput): Metadata => ({
  title,
  description,
  alternates: { canonical: path },
  openGraph: { title, description, url: path, images: ['/og.jpg'] },
  twitter: { card: 'summary_large_image', title, description },
});
```

- JSON-LD is a plain `<script>` with `<` escaped:

```tsx
<script
  type="application/ld+json"
  dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd).replace(/</g, '\\u003c') }}
/>
```

## 5. Verify

- `rg "'use client'" <route folder>` matches only leaf components, never `page.tsx`, `layout.tsx`, or a section.
- `rg "export default" <route folder>` matches only Next.js file conventions.
- Every section that loads data has a loading, empty, and error state.
- The rendered `<title>` goes through the root template, and the canonical URL is absolute.
- The route appears in navigation and `sitemap.ts` without a second hardcoded path.
- The repo's `typecheck` and `lint` scripts pass.
