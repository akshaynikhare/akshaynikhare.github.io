---
title: "Mono-Repo + Multi-Repo: How We Structured 6 Apps Across 4 Repositories"
date: 2026-05-23 13:00:00 +0530
categories: [Engineering, Architecture]
tags: [monorepo, multi-repo, ci-cd, mobile, react-native, architecture]
description: We didn't choose monorepo or multi-repo. We chose both — deliberately. Here's the structure, the reasoning, and where the seams caused friction.
---

Most teams treat "monorepo vs multi-repo" as a binary choice. Pick one, commit, move on. We ended up with a hybrid, and it turned out to be the right call — not out of indecision, but because our apps have genuinely different deployment and ownership characteristics.

Here's what we built, why, and what it costs.

## The System

The platform consists of six applications:

| App | Type | Primary Users |
|-----|------|---------------|
| REST API backend | Node.js + TypeScript | — (consumed by all apps) |
| Society dashboard | React web app | Society managers, admins, accountants |
| Company admin panel | React web app | Internal operations |
| Marketing website | Next.js | Public |
| Resident mobile app | React Native (Expo) | Residents |
| Guard mobile app | React Native (Expo) | Security personnel |

All six apps talk to the same API. But they have very different deployment cycles, team ownership, and testing requirements.

## The Structure: 4 Repositories

```
repo: main-platform (monorepo)
├── api/           — Express + Prisma backend
├── web-society/   — Society dashboard
├── web-admin/     — Company admin panel
└── web-marketing/ — Marketing site

repo: mobile-resident  — Resident app (React Native)
repo: mobile-guard     — Guard app (React Native)
repo: mobile-staff     — Society staff mobile app (React Native)
```

The web apps and the API live together in one monorepo. The three mobile apps each have their own repository.

## Why Split Mobile From Web?

The driving factor was **deployment cadence and review process**.

Web apps deploy on push — merge to main, CI builds, CDN updated within minutes. The feedback loop is fast, rollbacks are instant, and there's no approval gate between code and production.

Mobile apps go through app store review. A release cycle includes building a release APK, submitting to Google Play (and Apple App Store), waiting for review, and then a staged rollout. The cadence is measured in days, not minutes. Mistakes are expensive to reverse — a bad release means submitting a patch, waiting again, and potentially having a broken version live for days.

Given that difference, mobile releases warrant a separate release workflow, separate versioning (`release-please` per repo), separate EAS build config, and a dedicated changelog. Bundling mobile with web would mean mobile releases dragging along unrelated web changes and making the release notes hard to read.

**Separate repos give each mobile app its own clean release history.**

## Why Keep the Backend and Web Apps Together?

Because they change together constantly.

Most features touch both the API and a web frontend. A new billing feature means a new API endpoint, a new React page, and updates to both. With the API and web apps in the same repo:

- One PR covers the full change
- The CI pipeline validates everything together
- Reviewing a PR gives you the complete picture
- No cross-repo issue tracking for basic feature work

If they were in separate repos, every feature PR would spawn linked PRs across two repositories — more coordination overhead for changes that are logically one unit.

## The Cross-Repo Challenge

The seam between repos is where friction lives.

When a new API endpoint or a modified contract affects mobile apps, you get cross-repo work: one PR in the main platform repo, one or more PRs in the mobile repos. We handle this with explicit cross-linking:

- The parent issue lives in the main repo
- Each mobile repo gets a child issue referencing back
- PRs on both sides link to the same parent issue

It works, but it requires discipline. The worst version of this is when someone merges an API change without checking whether the mobile apps need a corresponding update. To guard against that, breaking API changes go through a separate review step that explicitly asks: "which mobile app versions does this affect, and are those PRs in flight?"

## Branch Protection Across All Repos

All four repos have identical branch protection on `main`:

- No direct pushes
- At least one approving review required
- CI must pass before merge

This is non-negotiable. With a multi-repo setup, it's easy for protection rules to diverge between repos. Keeping them identical means the same habits apply regardless of which repo you're working in.

## Shared Dependencies and Drift

The main risk with multi-repo mobile apps is dependency drift. Each mobile repo has its own `package.json`. If `main-platform` bumps a shared utility, the mobile repos don't automatically follow.

Our mitigation: the Expo SDK version and key shared dependencies (React Native, TypeScript, Babel config) are pinned to the same versions across all mobile repos. We check this in a periodic sync script rather than trying to enforce it automatically — automatic enforcement across repos adds tooling complexity we haven't needed yet.

## What We'd Do Differently

**Better automation for cross-repo issues.** Manually linking parent and child issues works until someone forgets. A GitHub Action that opens draft child issues in mobile repos when a label is applied to a main-repo issue would remove that failure mode.

**Shared TypeScript types package.** API response types are currently duplicated between the backend, web apps, and mobile apps. A shared `@platform/types` internal package published to a private registry (or referenced via workspace) would make type drift detectable at compile time. We've planned this but haven't needed it badly enough to prioritize it yet.

**Document the structure early.** The hybrid structure is intuitive once you understand the reasoning, but onboarding a new contributor requires explaining why mobile lives in separate repos. A `REPOS.md` at the root of the main repo — linking out and explaining the structure — saves the first 10 minutes of every onboarding conversation.

## The Summary

| Characteristic | Monorepo (web + API) | Multi-repo (mobile) |
|---------------|---------------------|---------------------|
| Deployment cadence | Continuous (minutes) | Store review (days) |
| Typical change scope | Multi-file, multi-layer | Self-contained per app |
| Release notes | Combined | Per app |
| PR complexity | High — covers whole feature | Focused |
| Cross-team coordination | Implicit (same repo) | Explicit (issue links) |

Monorepos reduce coordination overhead for tightly coupled code. Multi-repo reduces noise for code that releases independently. The answer isn't one or the other — it's matching your repo structure to your actual deployment and ownership boundaries.
