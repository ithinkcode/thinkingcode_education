# Project Prompt Template

A fill-in-the-blank template for generating full-stack applications. Copy this file, replace the bracketed sections, and delete the guidance comments.

---

# [Project Name] — [One-Line Description]

Generate a complete [monolith / split frontend+backend] application for [what it does in 2 sentences]. [Target users] can [core action 1], [core action 2], and [core action 3].

---

## 1. Technology Stack

<!-- Choose your stack. Be explicit about every major dependency. -->

| Layer | Choice |
|-------|--------|
| Framework | [e.g., Next.js 14+ App Router / Express.js + Next.js split] |
| Language | [e.g., TypeScript strict mode / Java 21] |
| Database | [e.g., PostgreSQL via Prisma / SQLite via Prisma / Spring Data JPA] |
| Styling | [e.g., Tailwind CSS v3 with custom theme] |
| Auth | [e.g., JWT in httpOnly cookies / JWT in Authorization header / Spring Security] |
| Validation | [e.g., Zod / Jakarta Bean Validation] |
| Icons | [e.g., Lucide React] |
| Toasts | [e.g., Sonner] |
| Fonts | [e.g., Playfair Display (headings) + Source Sans Pro (body) via next/font/google] |
| Other | [e.g., react-markdown, Framer Motion, Zustand] |

### Constraints
<!-- These prevent the AI from defaulting to generic patterns -->
- **No component libraries** (no shadcn, MUI, Chakra) — all UI custom-built
- **No Axios** — use native `fetch` with typed wrapper
- **No AI/LLM calls** — all features are deterministic
- [Add any project-specific constraints]
- **Responsive** — desktop primary, must work on mobile
- **Accessibility** — heading hierarchy, form labels, focus visible states

---

## 2. Database Schema

<!-- Write the ACTUAL schema, not a description of it.
     Use Prisma, SQL, or your ORM's schema syntax. -->

```prisma
model User {
  id            String   @id @default(cuid())
  name          String
  email         String   @unique
  passwordHash  String
  // ... add all fields
  createdAt     DateTime @default(now())
}

model [MainEntity] {
  id        String   @id @default(cuid())
  // ... define every field with types
  // ... define relationships
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

<!-- For each model, specify:
     - All fields with types
     - Relationships (@relation, foreign keys)
     - Unique constraints (@@unique)
     - Indexes if needed (@@index)
     - JSON fields if storing flexible data
-->

---

## 3. Authentication

<!-- Define the complete auth flow. Don't just say "add auth." -->

### Flow
1. **Register**: [describe registration: fields, what happens, where user goes next]
2. **Login**: [describe login: fields, token generation, storage]
3. **Logout**: [describe logout: clear tokens, redirect where]
4. **Token refresh**: [describe refresh strategy or say "not applicable"]

### Token Details
- Type: [JWT / session]
- Storage: [httpOnly cookie / localStorage / Authorization header]
- Expiration: [e.g., access: 15min, refresh: 7 days]
- Contents: [e.g., { userId: string }]

### Protected Routes
- [List all routes that require authentication]
- [Describe redirect behavior for unauthenticated users]

---

## 4. API Endpoints

<!-- Use tables. Every endpoint. No exceptions. -->

### Response Format
- Success: `{ data: T }`
- Error: `{ error: string, statusCode: number }`

### [Group 1: Auth]

| Method | Route | Body | Response | Auth |
|--------|-------|------|----------|------|
| POST | `/api/auth/register` | `{ name, email, password }` | `{ data: { user, token } }` | No |
| POST | `/api/auth/login` | `{ email, password }` | `{ data: { user, token } }` | No |
| GET | `/api/auth/me` | — | `{ data: User }` | Yes |

### [Group 2: Main Resource]

| Method | Route | Body | Response | Auth |
|--------|-------|------|----------|------|
| GET | `/api/[resources]` | Query: `?page=1&limit=10` | `{ data: Resource[] }` | [Yes/No] |
| POST | `/api/[resources]` | `{ field1, field2 }` | `{ data: Resource }` | Yes |
| GET | `/api/[resources]/[id]` | — | `{ data: Resource }` | [Yes/No] |
| PUT | `/api/[resources]/[id]` | `{ field1, field2 }` | `{ data: Resource }` | Yes |
| DELETE | `/api/[resources]/[id]` | — | `204` | Yes |

<!-- Add more groups as needed. For each endpoint, specify:
     - Query parameters
     - Request body shape with field types
     - Response shape
     - Auth requirements (no auth / any authenticated user / owner only / admin only)
     - Business logic (e.g., "slug auto-generated from title")
-->

---

## 5. Pages & Layouts

<!-- Describe each page top-to-bottom as if walking through a Figma mockup. -->

### Root Layout
- [Font loading]
- [Providers: auth, theme, toast]
- [Global elements: noise overlay, analytics]

### Navigation
- [Position: fixed top / sidebar / both]
- [Left content: logo, nav links]
- [Right content: user menu, CTAs]
- [Mobile behavior: hamburger menu, bottom nav]

### Landing Page (`/`)
- **Hero**: [headline text, tagline, CTA button]
- **Features section**: [number of cards, what each shows]
- **Social proof / stats**: [if applicable]
- **Footer**: [links, copyright]

### Login (`/login`) & Register (`/register`)
- [Layout: centered card / split screen]
- [Fields for each form]
- [Validation behavior]
- [Link between login/register]
- [Redirect destination on success]

### Dashboard (`/dashboard`) — Protected
- [Stats/summary section at top]
- [Main content: list, grid, table]
- [Actions: create new, filter, sort]
- [Empty state when no data]

### [Detail Page] (`/[resource]/[id]`)
- [Header: title, metadata, actions]
- [Main content area]
- [Sidebar or secondary info]
- [Related items or activity]

### [Editor/Form Page] (`/[resource]/new` or `/edit/[id]`)
- [Form layout]
- [Field types and validation]
- [Save/submit behavior]
- [Cancel behavior]

<!-- For each page, always include:
     - What it looks like with data
     - What it looks like WITHOUT data (empty state)
     - Mobile layout differences
     - Loading state
-->

---

## 6. Design Tokens & Styling

### Design Direction — "[Name]"
[2-3 sentences describing the aesthetic. Include reference points.]

### Colors
```
Background:      #______
Surface:         #______
Border:          #______
Text primary:    #______
Text secondary:  #______
Accent:          #______
Accent hover:    #______
Accent light:    #______
Success:         #______
Error:           #______
Warning:         #______
```

### Typography
- Headings: [Font name], [serif/sans] — weights [400, 600, 700]
- Body: [Font name], [serif/sans] — weights [400, 500, 600]
- Code: [Font name or system mono]
- Loading: [next/font/google / self-hosted]
- **Do NOT use**: [Inter alone, system defaults, etc.]

### Component Patterns
<!-- Define with actual Tailwind classes -->

**Cards:**
```
[bg/border/padding/radius/shadow/hover classes]
```

**Buttons:**
- Primary: `[classes]`
- Secondary: `[classes]`
- Ghost: `[classes]`
- Danger: `[classes]`

**Inputs:**
```
Default: [classes]
Focus: [classes]
Error: [classes]
```

**Badges/Pills:**
```
[Pattern for status indicators, tags, labels]
```

### Animations
- [What to animate and how]
- [Duration guidelines]
- [Must respect prefers-reduced-motion]

---

## 7. Project Structure

<!-- Write the exact file tree. Don't leave it to chance. -->

```
project-name/
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── globals.css
│   │   ├── [page]/page.tsx
│   │   └── api/
│   │       └── [route]/route.ts
│   ├── components/
│   │   ├── ui/            # Reusable primitives
│   │   ├── [feature]/     # Feature-specific components
│   │   └── layout/        # Navbar, Footer, Sidebar
│   ├── hooks/
│   ├── lib/               # Utilities, API client, auth helpers
│   ├── schemas/           # Zod validation schemas
│   ├── stores/            # Zustand stores (if needed)
│   └── types/
├── .env.example
├── .gitignore
├── package.json
└── [config files]
```

---

## 8. Environment Variables

### `.env.example`
```
DATABASE_URL=[connection string template]
JWT_SECRET=your-secret-key-change-in-production
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 9. Seed Data

<!-- Be specific. Use realistic names, titles, and content. -->

Create seed script that populates:

1. **Demo user**: [Full name] ([email] / [password])
   - [Bio or profile details]

2. **Sample data** ([number] [resources]):
   - "[Title 1]" — [brief description, key attributes]
   - "[Title 2]" — [brief description, key attributes]
   - "[Title 3]" — [brief description, key attributes]

<!-- Each piece of seed data should be realistic —
     real names, real titles, real content.
     No "Test Post" or "Lorem ipsum." -->

---

## 10. Configuration

### `package.json` scripts
```json
{
  "dev": "[dev command]",
  "build": "[build command]",
  "start": "[start command]",
  "lint": "[lint command]",
  "db:push": "[db push command]",
  "db:seed": "[seed command]"
}
```

### `.gitignore`
```
node_modules/
.next/
.env
.env.local
[database files if SQLite]
.DS_Store
```

---

## 11. Final Checks

Before delivering, verify:
- [ ] Dev server starts without errors
- [ ] Database initializes and seed data loads
- [ ] User registration and login work end to end
- [ ] All CRUD operations function correctly
- [ ] [Feature-specific check 1]
- [ ] [Feature-specific check 2]
- [ ] [Feature-specific check 3]
- [ ] Mobile layout is usable
- [ ] Forms show validation errors correctly
- [ ] Empty states render without errors
- [ ] Protected routes redirect unauthenticated users
