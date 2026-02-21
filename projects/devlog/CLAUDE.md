# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**DevLog** — a developer blogging platform. Users sign up, write posts in Markdown with live preview, and publish them on a personal public blog with RSS feed support. A single full-stack Next.js application — no separate backend service.

## Architecture

- **Full-stack Next.js 14+ App Router** monolith deployed on **Vercel**
- API routes (`app/api/`) handle auth, CRUD, and RSS generation
- Prisma ORM with **SQLite** — zero-config, file-based database
- JWT auth stored in httpOnly cookies (not Authorization headers)
- No external services — everything runs in one process

## Tech Stack

- Next.js 14+ App Router, TypeScript strict mode
- Prisma + SQLite (`dev.db` file)
- Tailwind CSS v3 with custom design tokens
- `react-markdown` + `remark-gfm` + `rehype-highlight` for rendering
- `@uiw/react-md-editor` or custom textarea with preview pane
- React Hook Form + Zod validation
- JWT (`jsonwebtoken`) in httpOnly cookies
- Lucide React icons, Sonner toasts
- Fonts: Newsreader (serif headings) + Inter (body) via `next/font`

## Commands

```bash
npm run dev          # next dev
npm run build        # next build
npm run lint         # next lint
npm run db:push      # prisma db push
npm run db:seed      # tsx prisma/seed.ts
npm run db:studio    # prisma studio
```

## Routes

### Pages
- `/` — public feed of recent published posts
- `/login`, `/register` — auth pages
- `/dashboard` — my posts (drafts + published), stats
- `/write` — markdown editor with live preview
- `/edit/[id]` — edit existing post
- `/[username]` — public author profile + post list
- `/[username]/[slug]` — public post page (SSR, SEO-optimized)

### API Routes
- `POST /api/auth/register|login|logout`, `GET /api/auth/me`
- `GET|POST /api/posts`, `GET|PUT|DELETE /api/posts/[id]`
- `GET /api/tags`
- `GET /api/[username]/feed.xml` — RSS feed

## Database

Three Prisma models: `User`, `Post`, `Tag` (many-to-many via implicit relation).

- `User`: id, name, email, passwordHash, bio, slug, avatarBase64, createdAt
- `Post`: id, title, slug, content (markdown), excerpt, coverImageBase64, published, publishedAt, readingTimeMin, authorId, createdAt, updatedAt
- `Tag`: id, name, slug
- Implicit `_PostToTag` join table

Profile photos and cover images stored as base64 in SQLite. No external file storage.

## Design Direction — "Clean Editorial"

Light mode default. Think Medium meets dev.to — content-first, generous whitespace, large readable text.

- Background: `#fafaf9` (warm white), cards: `#ffffff`
- Accent: indigo `#4f46e5`, hover: `#4338ca`
- Text: `#1c1917` primary, `#78716c` secondary
- Serif headings (Newsreader) + sans body (Inter) — dramatic size contrast
- Subtle shadows, no harsh borders
- Animations: minimal, content-focused transitions only
- Syntax highlighting: `github-light` theme in code blocks

## Key Features

- **Live Markdown Preview**: side-by-side or toggle editor/preview
- **Reading Time**: auto-calculated from word count (~200 WPM)
- **SEO**: dynamic meta tags, Open Graph, structured data on post pages
- **RSS**: per-author XML feed at `/[username]/feed.xml`
- **Syntax Highlighting**: fenced code blocks rendered with language-aware highlighting
- **Excerpts**: auto-generated from first 160 chars or manually set
- **Tags**: create-on-write, filterable on feed page

## Critical Constraints

- **No AI/LLM calls** — all features are deterministic
- **No external services** — SQLite file DB, base64 images, no S3/Cloudinary
- **No component libraries** (no shadcn, MUI, Chakra) — all UI custom-built
- **No Axios** — use native `fetch` in client components, direct Prisma calls in server components
- **Monolith** — no separate backend; API routes + pages in one Next.js app
- **Responsive** — desktop primary, must work on mobile
- **Accessibility** — proper heading hierarchy, form labels, focus states

## Environment Variables

```
DATABASE_URL="file:./dev.db"
JWT_SECRET=your-secret-key-change-in-production
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

## Seed Data

Demo author `demo@devlog.dev` / `demo1234` with 3-4 sample published posts covering different topics (JavaScript, system design, career advice) with varied tags and code blocks.
