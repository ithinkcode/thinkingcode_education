# Claude Code Prompt — Portfolio Platform Frontend (Vercel Deployment)

## Context

You are building the **frontend** for **FolioForge** — a portfolio generation platform demonstrated live in an education seminar. Students and graduates log in, walk through a guided onboarding wizard (answering questions about themselves or pasting resume text), and the platform generates a stunning, shareable portfolio page. They can then share a public link or download a beautifully designed PDF.

This frontend is a **Next.js 14+ App Router** application deployed on **Vercel**. It consumes a REST API backend (documented below). The design must be extraordinary — this is shown live to an audience of students and professionals. It must make jaws drop.

---

## Technology Stack (Mandatory)

- **Framework:** Next.js 14+ with App Router (TypeScript, strict mode)
- **Styling:** Tailwind CSS v3 with a fully custom design token system via `tailwind.config.ts` — custom colors, fonts, spacing scale, animations
- **Animations:** Framer Motion for page transitions, staggered reveals, micro-interactions, and the onboarding wizard flow
- **Icons:** Lucide React
- **Forms:** React Hook Form + Zod validation (mirror backend schemas)
- **HTTP Client:** Native `fetch` with a typed API client wrapper (no Axios)
- **State Management:** React Context for auth state, Zustand for portfolio builder state
- **Fonts:** Self-hosted via `next/font` — use **Clash Display** (display/headings) and **Satoshi** (body text) from Fontshare, or **General Sans** + **Cabinet Grotesk**. NO Inter, Roboto, or system fonts.
- **PDF Download:** Call backend `/api/portfolio/pdf` endpoint and trigger browser download
- **Toast Notifications:** Sonner

---

## Design Direction — "Editorial Luxury"

This is NOT a generic SaaS dashboard. The aesthetic is **editorial luxury meets developer portfolio** — think the intersection of a high-end design magazine and a premium developer tool.

### Design Principles

1. **Typography-First:** Headlines are enormous, expressive, and demand attention. Body text is perfectly spaced and readable. Use dramatic size contrast between headings and body (e.g., 72px headline / 16px body).

2. **Dark Mode Default:** The application default is a rich, deep dark theme — not pure black (#000), but layered darks with subtle warmth: backgrounds in `#0a0a0f`, cards in `#12121a`, borders in `#1e1e2e`. Accent color: a warm gold/amber `#d4a853` used sparingly for CTAs, highlights, and hover states. Secondary accent: a muted sage `#7c9a82`.

3. **Spatial Drama:** Generous whitespace. Asymmetric layouts where appropriate. Full-bleed hero sections. Content that breathes. The onboarding wizard should feel like stepping through rooms, not clicking through forms.

4. **Texture & Depth:** Subtle noise/grain overlay on backgrounds (CSS-only using SVG filters or pseudo-elements). Soft glows behind key UI elements. Layered card elevation with nuanced shadows. Glass-morphism used sparingly on modals and overlays.

5. **Motion Philosophy:** Every transition is intentional. Page loads use staggered reveals (elements cascade in from bottom with slight delays). Route transitions use smooth crossfades. The onboarding wizard steps slide horizontally with spring physics. Buttons have tactile press states (scale down on click). Progress indicators animate fluidly.

6. **Portfolio Themes:** The generated portfolios themselves support 3 visual themes that the user selects during onboarding:
   - **Obsidian** — dark, gold accents, dramatic typography (default)
   - **Arctic** — clean whites, ice-blue accents, crisp and minimal
   - **Ember** — warm cream backgrounds, burgundy/terracotta accents, editorial feel

---

## Page Structure & Routes

### Public Routes (No auth)
| Route | Page | Description |
|-------|------|-------------|
| `/` | Landing Page | Hero with animated headline, feature showcase, CTA to register. Should feel like a premium product launch page. |
| `/login` | Login | Email + password form with smooth validation states. Link to register. |
| `/register` | Register | First name, last name, email, password form. Auto-login after registration. |
| `/p/:slug` | Public Portfolio | The shareable portfolio page. Server-side rendered for SEO. Fetches from `/api/public/portfolio/:slug`. Render differently based on the portfolio's `theme` field. Include a floating "Download PDF" button and a "Built with FolioForge" watermark/badge. |

### Protected Routes (Auth required — redirect to `/login` if no token)
| Route | Page | Description |
|-------|------|-------------|
| `/onboarding` | Portfolio Builder Wizard | Multi-step guided flow (see below). Shown if user has no portfolio yet. |
| `/dashboard` | Dashboard | Overview of portfolio with preview, stats (published status, share link), quick edit access. |
| `/edit` | Portfolio Editor | Full edit interface for all portfolio sections. Tabbed or sectioned layout. |
| `/preview` | Portfolio Preview | Full preview of how the public portfolio will look. Toggle between themes. |

---

## Landing Page — Design Specification

The landing page is the first thing the seminar audience sees. It must be extraordinary.

**Hero Section:**
- Full viewport height
- Animated headline: "Your Story. Professionally Told." — each word animates in sequentially with a spring ease
- Subtitle: "Answer a few questions. Get a portfolio that opens doors."
- Two CTAs: "Get Started Free" (gold/primary, large) and "See Example" (ghost/outline)
- Background: subtle animated gradient mesh (CSS animation, slow-moving blobs of muted color behind a noise texture)
- A floating, parallax-responsive mockup of an example portfolio (use a styled card with real content, not an image)

**Features Section:**
- 3 feature cards in an asymmetric grid
- Each card has an icon, bold headline, and short description
- Cards: "Guided Builder" (walk through questions), "Instant PDF" (download professional resume), "Share Anywhere" (public link)
- Cards animate in on scroll using Framer Motion's `whileInView`

**Social Proof / Demo Section:**
- "See what FolioForge creates" with an embedded interactive mini-preview of a sample portfolio
- Or a before/after: "From resume text..." → "...to this" with a polished portfolio card

**Footer:**
- Minimal. "Built for the next generation of engineers." + Thinking Code Technologies branding

---

## Onboarding Wizard — The Core Experience

This is the heart of the product. The wizard transforms answers into a portfolio. It should feel premium, guided, and encouraging — like a conversation, not a form.

### Wizard Architecture
- Use Zustand store to persist wizard state across steps (survives page refreshes via `localStorage`)
- Horizontal step indicator at the top showing progress with animated fill
- Each step slides in from the right with Framer Motion `AnimatePresence`
- "Back" and "Continue" navigation with validation per step
- Final step shows a live preview before publishing

### Steps

**Step 0 — Welcome & Resume Shortcut**
- "Let's build your portfolio. You can answer questions step by step, or paste your resume to get a head start."
- Large textarea: "Paste your resume text here (optional)"
- If text is pasted, call `POST /api/portfolio/parse-resume` and pre-fill subsequent steps with parsed data
- "Start Fresh" button to skip

**Step 1 — Identity**
- First name, last name (pre-filled from registration)
- Professional headline (text input with placeholder examples that rotate: "Full-Stack Developer", "Systems Engineer & Problem Solver", "Creative Technologist")
- Profile photo upload (drag-and-drop zone with preview, converts to base64)

**Step 2 — Your Story**
- "Tell us about yourself in 2-3 paragraphs. What drives you? What do you bring to a team?"
- Large textarea with character count
- Helper prompt below: "Think about: your technical philosophy, what excites you about building software, and what makes you different."

**Step 3 — Skills**
- Add skills with proficiency level (1-5 stars or slider)
- Category selector: "Frontend", "Backend", "DevOps", "Data", "Design", "Other"
- Chips/tags UI that animates as skills are added
- "Suggested skills" quick-add buttons based on common tech skills

**Step 4 — Experience**
- Add work experiences (company, role, start/end dates, highlight bullets)
- Each entry is a collapsible card
- "Add another" button with smooth expand animation
- Support for "Present" as end date

**Step 5 — Education**
- Institution, degree, field of study, graduation year
- Achievements/honors as tag inputs
- Multiple entries supported

**Step 6 — Projects (Make them shine)**
- Project title, description, tech stack (tag input), live URL, repo URL
- Optional project image upload (base64)
- This section should feel exciting — "Show off your best work"
- Cards with rich preview showing entered data

**Step 7 — Extras**
- Certifications (name, issuer, year)
- Languages (language name, proficiency level)
- Both as simple repeatable forms

**Step 8 — Contact & Links**
- Email (pre-filled), LinkedIn URL, GitHub URL, Twitter/X URL, personal website
- Input validation for URL formats

**Step 9 — Theme Selection & Preview**
- Choose between Obsidian, Arctic, and Ember themes
- Show a LIVE preview of their portfolio in each theme (3 clickable cards, each showing a miniaturized version of their data rendered in that theme)
- Selected theme has a glowing border animation

**Step 10 — Launch**
- Full portfolio preview at actual size
- Custom slug editor (auto-generated from name, editable)
- Share URL preview: `folioforge.dev/p/your-slug`
- Two CTAs: "Publish Portfolio" (calls publish endpoint) and "Save as Draft"
- Confetti animation on publish (use canvas-confetti library)

---

## Public Portfolio Page (`/p/:slug`) — Showcase Design

This is what gets shared. It must be portfolio-of-the-year quality.

### Layout (Obsidian Theme — default)
- **Hero:** Full-width decorative gradient banner (`h-52 sm:h-56`, `bg-gradient-to-br from-[#d4a853]/15 via-[#1a1a2e] to-[#0a0a0f]`) with the profile photo enlarged to 144px (`h-36 w-36`), overlapping the banner bottom edge via `-mt-20`, and a `border-4` in the page background color so it cleanly cuts into the banner. Name in massive display type and headline below the photo
- **Summary:** Clean two-column — summary text on the left, contact links as icons on the right
- **Skills:** Visual skill bars or a radar chart rendered with CSS (no chart library needed). Skills grouped by category with animated fill-on-scroll
- **Experience:** Timeline layout with vertical line, dots for each role, dates on alternating sides. Each entry expands on click/hover to show highlights
- **Projects:** Masonry or bento grid of project cards. Each card shows title, tech stack as small tags, and description. Cards have hover states with subtle lift and glow
- **Education:** Clean cards with institution logo placeholder (initials in a colored circle)
- **Certifications & Languages:** Compact grid of badge-style items
- **Footer:** "Download PDF" button (prominent) + "Built with FolioForge" subtle branding

All sections animate into view on scroll using Framer Motion's `whileInView` with staggered children.

### Theme Variations
- **Arctic:** White backgrounds, light gray cards, ice-blue (#4da8da) accent color, clean sans-serif typography, sharp borders instead of glows. Hero gradient banner: `from-[#4da8da]/20 via-[#e0f0fa] to-[#f8fafc]`. Photo border: `border-[#f8fafc]`
- **Ember:** Warm cream (#faf5ef) background, burgundy (#8b2635) headings, terracotta (#c46a4a) accents, serif display font (Playfair Display), subtle paper texture overlay. Hero gradient banner: `from-[#8b2635]/15 via-[#f0e0d0] to-[#faf5ef]`. Photo border: `border-[#faf5ef]`

---

## API Client

Create a typed API client at `lib/api.ts`:

```typescript
// Base URL from environment: NEXT_PUBLIC_API_URL
// All methods return typed responses
// Automatically attaches JWT from cookie/localStorage
// Handles 401 by clearing auth state and redirecting to /login
// Handles token refresh transparently

const api = {
  auth: {
    register(data: RegisterInput): Promise<AuthResponse>,
    login(data: LoginInput): Promise<AuthResponse>,
    refresh(): Promise<AuthResponse>,
    me(): Promise<User>,
  },
  portfolio: {
    get(): Promise<Portfolio>,
    create(data: CreatePortfolioInput): Promise<Portfolio>,
    update(data: UpdatePortfolioInput): Promise<Portfolio>,
    patch(data: Partial<Portfolio>): Promise<Portfolio>,
    publish(): Promise<Portfolio>,
    unpublish(): Promise<Portfolio>,
    delete(): Promise<void>,
    parseResume(text: string): Promise<Partial<Portfolio>>,
    downloadPdf(): Promise<Blob>,
  },
  public: {
    getPortfolio(slug: string): Promise<Portfolio>,
  },
}
```

---

## Auth Implementation

- Store tokens in `httpOnly` cookies where possible, fallback to `localStorage` with memory cache
- Auth context provider wrapping the app in `layout.tsx`
- `useAuth()` hook returning `{ user, isLoading, isAuthenticated, login, register, logout }`
- Protected route wrapper component that redirects to `/login` with return URL
- On app load, call `/api/auth/me` to validate existing token

---

## Project Structure

```
frontend/
├── src/
│   ├── app/
│   │   ├── layout.tsx            # Root layout with providers, fonts, global styles
│   │   ├── page.tsx              # Landing page
│   │   ├── login/page.tsx
│   │   ├── register/page.tsx
│   │   ├── p/[slug]/page.tsx     # Public portfolio (SSR)
│   │   ├── (protected)/          # Route group with auth guard layout
│   │   │   ├── layout.tsx        # Auth check wrapper
│   │   │   ├── onboarding/page.tsx
│   │   │   ├── dashboard/page.tsx
│   │   │   ├── edit/page.tsx
│   │   │   └── preview/page.tsx
│   │   └── globals.css
│   ├── components/
│   │   ├── ui/                   # Reusable primitives (Button, Input, Card, Modal, etc.)
│   │   ├── landing/              # Landing page sections
│   │   ├── onboarding/           # Wizard steps (Step0.tsx through Step10.tsx)
│   │   ├── portfolio/            # Portfolio rendering components (themed)
│   │   │   ├── PortfolioView.tsx # Main portfolio renderer
│   │   │   ├── themes/
│   │   │   │   ├── ObsidianTheme.tsx
│   │   │   │   ├── ArcticTheme.tsx
│   │   │   │   └── EmberTheme.tsx
│   │   │   ├── sections/         # HeroSection, SkillsSection, ExperienceSection, etc.
│   │   │   └── PdfDownloadButton.tsx
│   │   ├── auth/
│   │   │   ├── AuthProvider.tsx
│   │   │   ├── LoginForm.tsx
│   │   │   ├── RegisterForm.tsx
│   │   │   └── ProtectedRoute.tsx
│   │   └── shared/
│   │       ├── Navbar.tsx
│   │       ├── Footer.tsx
│   │       ├── GrainOverlay.tsx  # Subtle noise texture overlay
│   │       └── AnimatedPage.tsx  # Page transition wrapper
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   └── usePortfolio.ts
│   ├── stores/
│   │   └── onboardingStore.ts   # Zustand store for wizard state
│   ├── lib/
│   │   ├── api.ts               # Typed API client
│   │   ├── fonts.ts             # next/font configurations
│   │   └── utils.ts             # cn() helper, formatDate, etc.
│   ├── schemas/
│   │   ├── auth.ts              # Zod schemas matching backend
│   │   └── portfolio.ts
│   └── types/
│       └── index.ts             # Shared TypeScript interfaces
├── public/
│   └── og-image.png             # Open Graph image (create a simple branded one)
├── next.config.js
├── tailwind.config.ts
├── tsconfig.json
├── package.json
├── vercel.json
└── README.md
```

---

## Vercel Configuration

**`vercel.json`:**
```json
{
  "framework": "nextjs",
  "buildCommand": "npm run build",
  "env": {
    "NEXT_PUBLIC_API_URL": "@api-url"
  }
}
```

**Environment Variables for Vercel:**
```
NEXT_PUBLIC_API_URL=https://your-railway-backend.up.railway.app
NEXT_PUBLIC_APP_URL=https://your-vercel-domain.vercel.app
```

---

## Environment Variables (.env.local.example)

```env
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## Tailwind Config Highlights

```typescript
// tailwind.config.ts — key customizations
{
  theme: {
    extend: {
      colors: {
        surface: {
          DEFAULT: '#0a0a0f',
          raised: '#12121a',
          overlay: '#1a1a28',
          border: '#1e1e2e',
        },
        accent: {
          gold: '#d4a853',
          goldHover: '#e6bc6a',
          sage: '#7c9a82',
        },
        // Arctic theme
        arctic: { bg: '#f8fafc', card: '#ffffff', accent: '#4da8da' },
        // Ember theme
        ember: { bg: '#faf5ef', card: '#fff8f0', accent: '#8b2635', warm: '#c46a4a' },
      },
      fontFamily: {
        display: ['var(--font-clash-display)'],
        body: ['var(--font-satoshi)'],
      },
      animation: {
        'fade-up': 'fadeUp 0.6s ease-out',
        'slide-in': 'slideIn 0.5s ease-out',
        'glow-pulse': 'glowPulse 2s ease-in-out infinite',
      },
    },
  },
}
```

---

## Critical CSS Details

### Grain/Noise Overlay (GrainOverlay component)
```css
.grain-overlay::after {
  content: '';
  position: fixed;
  inset: 0;
  z-index: 9999;
  pointer-events: none;
  opacity: 0.03;
  background-image: url("data:image/svg+xml,..."); /* inline SVG noise */
  mix-blend-mode: overlay;
}
```

### Gradient Mesh Background (Landing Hero)
```css
.gradient-mesh {
  background:
    radial-gradient(ellipse at 20% 50%, rgba(212, 168, 83, 0.08) 0%, transparent 50%),
    radial-gradient(ellipse at 80% 20%, rgba(124, 154, 130, 0.06) 0%, transparent 50%),
    radial-gradient(ellipse at 50% 80%, rgba(212, 168, 83, 0.04) 0%, transparent 50%);
  animation: meshShift 20s ease-in-out infinite alternate;
}
```

---

## Performance Requirements

- Lighthouse score > 90 on all metrics
- Images lazy-loaded with blur placeholders
- Route-based code splitting (automatic with Next.js App Router)
- Fonts loaded with `display: swap` and preloaded
- Portfolio public pages are server-rendered (`generateMetadata` for SEO with Open Graph tags)

---

## Final Checks

After generating all code:

1. Run `npm install`
2. Run `npm run build` — fix all TypeScript and build errors
3. Run `npm run lint` — clean all lint issues
4. Verify all routes render without errors
5. Test the onboarding flow end-to-end mentally (check state management covers all steps)
6. Ensure the public portfolio page renders all 3 themes correctly
7. Initialize a git repository, create `.gitignore` (include `node_modules`, `.next`, `.env.local`), and commit with message: `feat: FolioForge frontend — portfolio platform UI`
8. Verify `vercel.json` is valid and the project structure matches Vercel's Next.js expectations

---

## IMPORTANT CONSTRAINTS

- **No component libraries** (no shadcn, no MUI, no Chakra). All UI components are custom-built. This is both a design choice and a teaching moment in the seminar — "you can build beautiful things from scratch."
- **No AI calls from the frontend.** Resume parsing goes through the backend API.
- **Responsive design** — must work beautifully on desktop (primary) and tablet/mobile.
- **Accessibility baseline** — proper heading hierarchy, form labels, focus states, color contrast.
- **Every animation must have `prefers-reduced-motion` respect** — wrap motion in a media query check.
