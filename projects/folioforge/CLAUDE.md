# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**FolioForge** — a portfolio generation platform for an education seminar. Users log in, complete a guided onboarding wizard (or paste resume text), and the system generates a shareable portfolio page with PDF download. The project is split into two independently deployed services: a backend API and a frontend app.

## Architecture

- **Backend (`backend/`):** Express.js REST API deployed on **Railway** via Docker. Handles auth, portfolio CRUD, PDF generation (Puppeteer), and resume parsing (heuristic, no AI/LLM).
- **Frontend (`frontend/`):** Next.js 14+ App Router deployed on **Vercel**. Consumes the backend API. Features a multi-step onboarding wizard and themed public portfolio pages.
- Communication is REST over HTTPS. JWT-based stateless auth (access + refresh tokens). No sessions.

## Backend

### Tech Stack
- Node.js 20+, TypeScript, Express.js
- PostgreSQL via Prisma ORM (`DATABASE_URL` from Railway)
- JWT auth (`bcryptjs` + `jsonwebtoken`), Zod validation
- Puppeteer for PDF generation (headless Chromium)
- `helmet`, `compression`, `express-rate-limit`, `morgan`

### Commands
```bash
npm run dev          # tsx watch src/index.ts
npm run build        # tsc
npm start            # node dist/index.js
npm run db:push      # prisma db push
npm run db:seed      # tsx prisma/seed.ts
npm run db:studio    # prisma studio
npm run lint         # eslint src/
npm run typecheck    # tsc --noEmit
```

### Key API Routes
- `POST /api/auth/register|login|refresh`, `GET /api/auth/me`
- `GET|POST|PUT|PATCH|DELETE /api/portfolio`, `POST /api/portfolio/publish|unpublish`
- `GET /api/portfolio/pdf` — streams PDF as `application/pdf`
- `POST /api/portfolio/parse-resume` — heuristic text parsing, no AI
- `GET /api/public/portfolio/:slug` — public, no auth
- `GET /api/health` — Railway health check

### Response Format
- Success: `{ data: T }`
- Error: `{ error: string, details?: any, statusCode: number }`

### Database
Two Prisma models: `User` and `Portfolio` (1:1 relation). Portfolio stores skills, experience, education, projects, certifications, languages, and contact as JSON columns. Profile photos stored as base64 in PostgreSQL (`bytea`). No external file storage.

### PDF Generation
Puppeteer renders a self-contained HTML string (inline CSS, embedded Google Fonts) into A4 PDF. Three themes: `obsidian` (charcoal + gold), `arctic` (white + ice-blue), `ember` (burgundy + cream). Two-column layout with sidebar. Uses `Playfair Display` + `Source Sans Pro`.

### Seed Data
Demo user `demo@folioforge.dev` / `demo1234` with a fully populated portfolio for "Alex Chen."

## Frontend

### Tech Stack
- Next.js 14+ App Router, TypeScript strict mode
- Tailwind CSS v3 with custom design tokens
- Framer Motion for animations
- React Hook Form + Zod, Zustand (wizard state), React Context (auth)
- Lucide React icons, Sonner toasts, canvas-confetti
- Fonts: Clash Display (headings) + Satoshi (body) via `next/font` — no Inter/Roboto/system fonts

### Commands
```bash
npm run dev          # next dev
npm run build        # next build
npm run lint         # next lint
```

### Routes
- Public: `/` (landing), `/login`, `/register`, `/p/:slug` (SSR portfolio)
- Protected (auth guard layout): `/onboarding` (10-step wizard), `/dashboard`, `/edit`, `/preview`

### Design Direction — "Editorial Luxury"
Dark mode default (`#0a0a0f` base, `#12121a` cards). Gold accent `#d4a853`, sage secondary `#7c9a82`. Typography-first with dramatic size contrast. Subtle grain/noise overlay. All animations respect `prefers-reduced-motion`.

### Portfolio Themes (same 3 as PDF)
Each theme's public portfolio hero has a decorative gradient banner (`h-52 sm:h-56`) with the profile photo (`h-36 w-36`) overlapping its bottom edge via `-mt-20` and a `border-4` in the page background color.
- **Obsidian:** dark + gold accents. Banner: `from-[#d4a853]/15 via-[#1a1a2e] to-[#0a0a0f]`
- **Arctic:** white + ice-blue `#4da8da`. Banner: `from-[#4da8da]/20 via-[#e0f0fa] to-[#f8fafc]`
- **Ember:** cream `#faf5ef` + burgundy `#8b2635` + terracotta `#c46a4a`. Banner: `from-[#8b2635]/15 via-[#f0e0d0] to-[#faf5ef]`

### API Client
Typed client at `lib/api.ts` using native `fetch`. Auto-attaches JWT, handles 401 redirect, transparent token refresh. Base URL from `NEXT_PUBLIC_API_URL`.

## Critical Constraints

- **No AI/LLM calls** anywhere — resume parsing is heuristic pattern matching only
- **No external file storage** — everything in PostgreSQL or ephemeral disk
- **No component libraries** (no shadcn, MUI, Chakra) — all UI is custom-built
- **No Axios** — use native `fetch` with typed wrapper
- **Stateless** — JWT-only auth, no sessions, horizontally scalable
- **Responsive** — desktop primary, must work on tablet/mobile
- **Accessibility** — heading hierarchy, form labels, focus states, color contrast

## Environment Variables

### Backend (`.env.example`)
```
DATABASE_URL, JWT_SECRET, JWT_REFRESH_SECRET, JWT_EXPIRES_IN (15m),
JWT_REFRESH_EXPIRES_IN (7d), PORT (3001), ALLOWED_ORIGINS, NODE_ENV, FRONTEND_URL
```

### Frontend (`.env.local.example`)
```
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_APP_URL=http://localhost:3000
```
