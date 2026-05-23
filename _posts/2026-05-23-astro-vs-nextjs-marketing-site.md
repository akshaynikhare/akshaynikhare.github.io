---
title: "Why We Chose Astro Over Next.js for Our Marketing Site"
date: 2026-05-23 11:00:00 +0530
categories: [Engineering, Frontend]
tags: [astro, nextjs, performance, ssr, static-site, web]
description: We chose Astro 5 over Next.js for a marketing site. No React, vanilla TypeScript, hand-rolled CSS. Here's the reasoning and what we'd do differently.
---

When we started building [mistyug.com](https://mistyug.com), the default choice would have been Next.js. It's what most teams reach for. We use React elsewhere in our stack. The ecosystem is mature.

We chose Astro instead. Here's why — and where the tradeoffs actually landed.

## The Context

A marketing site has a specific job: load fast, communicate clearly, rank well in search, and require minimal maintenance. It is not an application. Users don't log in. There's no client-side state to manage. The page content changes a handful of times per quarter.

That job description should immediately make you question whether you need a framework designed for building applications.

## What Next.js Would Have Given Us

Next.js excels when you need:

- Server-rendered pages that fetch data at request time
- Client-side interactivity within a consistent component model
- API routes colocated with your frontend
- A shared codebase with your React application

For a marketing site, exactly one of those applies: we do fetch some data at build time (project details, structured content). The rest — client-side interactivity, API routes, React components — we simply don't need.

Next.js would have shipped kilobytes of JavaScript runtime to users just to render static HTML that could have been generated at build time and served as a file.

## What Astro Gave Us

Astro's core idea is zero JavaScript by default. Pages are rendered at build time. No client-side hydration happens unless you explicitly request it. If a component needs interactivity, you opt into it per-component with `client:load` or `client:visible` — everything else ships as pure HTML and CSS.

For a marketing site, this is exactly right.

```astro
---
// Component script runs at build time only
const projects = await getProjects();
---

<section>
  {projects.map(p => (
    <article>
      <h2>{p.title}</h2>
      <p>{p.description}</p>
    </article>
  ))}
</section>

<!-- Zero JS shipped. This is static HTML. -->
```

The output is a directory of `.html` files and assets. Deployable anywhere. No Node.js runtime required. CDN-cacheable at the edge with no configuration.

## The Stack We Landed On

- **Framework:** Astro 5.x
- **Scripting:** Vanilla TypeScript in `<script>` blocks — no build step, no bundler configuration
- **Styles:** Hand-rolled CSS — one file, no framework, no utility classes
- **Icons:** Lucide (tree-shaken, no runtime overhead)
- **Deployment:** Static output, edge CDN

No React. No Tailwind. No component library.

This feels radical if you're used to application-scale tooling, but for a site with six pages and a handful of sections, it's the appropriate level of complexity.

## Lighthouse Scores After Launch

Marketing sites are judged on Core Web Vitals. Astro's static output with no hydration delivers near-perfect scores out of the box:

- **Performance:** 98
- **Accessibility:** 100
- **Best Practices:** 100
- **SEO:** 100

Getting equivalent scores from a Next.js project is possible, but it requires deliberate optimization work: eliminating unnecessary client components, configuring `next/font`, controlling what gets bundled. Astro's default is already optimized.

## Where Astro Falls Short

Honest tradeoffs:

**1. Island architecture has a learning curve.** When you do need interactivity — a form, a scroll animation, a sticky nav — you need to understand Astro's island model to know which `client:*` directive is appropriate. It's not hard, but it's new mental model if you're coming from React.

**2. Ecosystem is smaller.** Need a specific CMS integration or third-party component? You'll find more Next.js examples. Astro's ecosystem is growing fast, but Next.js wins on breadth.

**3. Sharing code with your React app takes deliberate effort.** If your component design system lives in React, you can use React components in Astro (`client:visible`), but it adds a layer of indirection. We kept the marketing site's component set small enough that this wasn't a problem.

**4. Not the right choice if you need a hybrid.** If your "marketing site" also has a blog with user comments, a dashboard behind auth, or pages that fetch live data per-request, Next.js is the better fit. Astro can do these things, but it's not where it shines.

## The Decision Framework

Use **Astro** when:
- Content is static or build-time generated
- Performance and simplicity are top priorities
- The site doesn't need a persistent client-side runtime
- You want a clean separation between content and application

Use **Next.js** when:
- You need per-request server rendering
- The site shares components with a React application
- You'll add authenticated routes or API endpoints
- You need real-time data or user-specific content

## Would We Make the Same Choice Again?

Yes. Mistyug.com loads in under a second on a 4G connection. Maintenance is minimal — changing a section means editing a `.astro` file, not tracing through a React component tree. The build output is a folder of files that can be dropped anywhere.

For a marketing site, the question isn't "why Astro?" It's "why not?"
