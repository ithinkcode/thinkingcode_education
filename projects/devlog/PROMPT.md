# DevLog — Full-Stack Next.js Developer Blogging Platform

Generate a complete full-stack developer blogging platform as a **single Next.js 14+ App Router monolith**. Users register, write Markdown posts with live preview and syntax highlighting, and publish them on a personal public blog with RSS feed support.

---

## 1. Technology Stack

| Layer | Choice |
|-------|--------|
| Framework | Next.js 14+ App Router (single monolith — API routes + pages) |
| Language | TypeScript strict mode everywhere |
| Database | SQLite via Prisma ORM (file: `./dev.db`) |
| Styling | Tailwind CSS v3 with custom theme |
| Markdown | `react-markdown` + `remark-gfm` + `rehype-highlight` |
| Editor | Custom split-pane: textarea + live preview (no WYSIWYG library) |
| Auth | JWT in httpOnly cookies (`jsonwebtoken` + `bcryptjs`) |
| Validation | Zod schemas, React Hook Form on client |
| Icons | Lucide React |
| Toasts | Sonner |
| Fonts | Newsreader (serif headings) + Inter (body) via `next/font/google` |

### Constraints
- **No AI/LLM calls** — all features are deterministic
- **No component libraries** (no shadcn, MUI, Chakra) — every component hand-built
- **No Axios** — use native `fetch` in client components; call Prisma directly in server components / API routes
- **No separate backend** — everything lives in one Next.js app
- **No external storage** — images are base64 in SQLite
- **Responsive** — desktop-first, must work on mobile
- **Accessibility** — proper heading hierarchy, form labels, focus visible states, color contrast ≥ 4.5:1

---

## 2. Database Schema (Prisma)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model User {
  id            String   @id @default(cuid())
  name          String
  email         String   @unique
  passwordHash  String
  bio           String?
  slug          String   @unique
  avatarBase64  String?
  createdAt     DateTime @default(now())
  posts         Post[]
}

model Post {
  id               String    @id @default(cuid())
  title            String
  slug             String
  content          String    // raw Markdown
  excerpt          String?   // first 160 chars or custom
  coverImageBase64 String?
  published        Boolean   @default(false)
  publishedAt      DateTime?
  readingTimeMin   Int       @default(1)
  author           User      @relation(fields: [authorId], references: [id])
  authorId         String
  tags             Tag[]
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  @@unique([authorId, slug])
}

model Tag {
  id    String @id @default(cuid())
  name  String @unique
  slug  String @unique
  posts Post[]
}
```

---

## 3. Authentication

Cookie-based JWT auth — **not** Authorization headers.

### Flow
1. **Register**: hash password with `bcryptjs`, create user, set JWT cookie, return user.
2. **Login**: verify credentials, set JWT cookie, return user.
3. **Logout**: clear the JWT cookie.
4. **Auth check**: middleware reads the `token` cookie, verifies with `jsonwebtoken`, attaches `userId` to the request.

### JWT Cookie
- Name: `token`
- `httpOnly: true`, `secure: true` in production, `sameSite: 'lax'`
- Expires in 7 days
- Contains: `{ userId: string }`

### Auth Middleware (for API routes)
Create a helper `getAuthUser(request)` that reads the cookie, verifies the JWT, and returns the user or `null`. Use in API routes:
```ts
const user = await getAuthUser(request);
if (!user) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
```

### Auth Context (client)
React context provider that:
- Calls `GET /api/auth/me` on mount to check session
- Exposes `user`, `login()`, `register()`, `logout()`, `isLoading`
- Redirects unauthenticated users away from `/dashboard`, `/write`, `/edit/*`

---

## 4. API Routes

All API routes live under `app/api/`. Return JSON with consistent shape:
- Success: `{ data: T }`
- Error: `{ error: string }` with appropriate status code

### Auth
| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/auth/register` | Create account, set cookie |
| POST | `/api/auth/login` | Verify credentials, set cookie |
| POST | `/api/auth/logout` | Clear cookie |
| GET | `/api/auth/me` | Return current user from cookie |

### Posts
| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/posts` | My posts (auth required). Query: `?status=draft\|published` |
| POST | `/api/posts` | Create post (auth required) |
| GET | `/api/posts/[id]` | Get single post (auth: own posts; public: published only) |
| PUT | `/api/posts/[id]` | Update post (auth, own only) |
| DELETE | `/api/posts/[id]` | Delete post (auth, own only) |
| GET | `/api/posts/published` | Public feed. Query: `?tag=slug&page=1&limit=10` |

### Tags
| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/tags` | All tags with post counts |

### RSS
| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/[username]/feed.xml` | RSS 2.0 XML for author's published posts |

### Post Body Shape
```ts
{
  title: string;            // required, 1-200 chars
  content: string;          // required, Markdown
  excerpt?: string;         // optional, auto-generated if empty
  coverImageBase64?: string;
  published?: boolean;      // default false
  tags?: string[];          // tag names — create if not exist
}
```

### Reading Time
Calculate on save: `Math.max(1, Math.ceil(wordCount / 200))`.

### Slug Generation
Post slugs: kebab-case from title. If duplicate for the same author, append `-2`, `-3`, etc.
User slugs: kebab-case from name at registration. If duplicate, append random 4-char suffix.

---

## 5. Pages & Layouts

### Root Layout (`app/layout.tsx`)
- Load Newsreader + Inter fonts
- `<AuthProvider>` wrapping the app
- `<Toaster>` (Sonner) for notifications
- Light background `#fafaf9`

### Navigation Bar
- Fixed top bar, white background with subtle bottom border
- Left: logo "DevLog" in serif font
- Right (unauthenticated): "Log in" and "Sign up" buttons
- Right (authenticated): "Write" button (primary CTA), user avatar dropdown (Dashboard, Settings, Logout)

### Landing / Public Feed (`/`)
- Hero section: "Where developers write." — large serif headline, subtle tagline, CTA to sign up
- Below hero: grid/list of recent published posts from all authors
- Each card: cover image (or gradient placeholder), title, excerpt, author name + avatar, date, reading time, tags
- Pagination at bottom
- Tag filter pills at top

### Login & Register (`/login`, `/register`)
- Centered card, clean form
- Email + password fields, submit button
- Link to the other page ("Don't have an account? Sign up")
- Show validation errors inline (Zod)
- Redirect to `/dashboard` on success

### Dashboard (`/dashboard`) — Protected
- "My Posts" heading with count
- Tabs: "Published" / "Drafts"
- Post list: title, status badge, date, reading time, tag chips, actions (edit, delete, view)
- "Write a post" CTA if no posts
- Stats at top: total posts, total published, total reading time across posts

### Write Post (`/write`) — Protected
- **Split-pane editor**: left = raw Markdown textarea, right = rendered preview
- Toggle button to switch between split view and fullscreen editor/preview
- Title input at top (large, borderless, serif font — like Medium)
- Tag input: comma-separated, auto-suggest from existing tags
- Cover image upload (converts to base64, shows preview)
- Bottom toolbar: "Save Draft" (secondary) and "Publish" (primary) buttons
- Auto-generate excerpt from first 160 chars of content if not manually set
- Calculate and display reading time live as user types

### Edit Post (`/edit/[id]`) — Protected
- Same editor as `/write`, pre-populated with post data
- Additional "Unpublish" button if currently published
- "Delete" button with confirmation modal

### Public Author Profile (`/[username]`)
- Author card: avatar, name, bio, join date, post count
- Grid of published posts (same card style as feed)
- RSS icon linking to `/[username]/feed.xml`

### Public Post (`/[username]/[slug]`)
- **SSR** with dynamic meta tags (title, description, og:image)
- Article layout: max-width `prose` container
- Cover image (full width, rounded)
- Title (large serif), author bar (avatar, name, date, reading time)
- Rendered Markdown content with:
  - GitHub Flavored Markdown (tables, task lists, strikethrough)
  - Syntax-highlighted code blocks with language label and copy button
  - Responsive images
  - Block quotes styled distinctively
- Tag chips at bottom
- Author card at bottom with link to profile
- "Back to feed" link

---

## 6. Design Tokens & Styling

### Colors
```
Background:     #fafaf9 (warm stone-50)
Surface:        #ffffff
Border:         #e7e5e4 (stone-200)
Text primary:   #1c1917 (stone-900)
Text secondary: #78716c (stone-500)
Accent:         #4f46e5 (indigo-600)
Accent hover:   #4338ca (indigo-700)
Accent light:   #eef2ff (indigo-50) — tag backgrounds
Success:        #16a34a
Error:          #dc2626
```

### Typography
- Headings: `Newsreader` serif, weights 400/600/700
- Body: `Inter` sans, weights 400/500/600
- Code: system monospace stack
- Base size: 16px body, 18px article content
- Article `h1`: 2.5rem, `h2`: 2rem, `h3`: 1.5rem — generous spacing

### Component Patterns
- **Cards**: white background, `rounded-xl`, `shadow-sm`, `border border-stone-200`, `hover:shadow-md` transition
- **Buttons**: `rounded-lg`, `font-medium`, `transition-colors`, `focus-visible:ring-2 ring-indigo-500 ring-offset-2`
  - Primary: indigo background, white text
  - Secondary: white background, stone border, stone text
  - Ghost: transparent, hover stone-100
- **Inputs**: `rounded-lg`, `border border-stone-300`, `focus:border-indigo-500 focus:ring-1`, padding `px-4 py-2.5`
- **Tag chips**: `rounded-full`, `bg-indigo-50 text-indigo-700`, `px-3 py-1`, `text-sm font-medium`
- **Status badges**: Draft = `bg-amber-100 text-amber-800`, Published = `bg-emerald-100 text-emerald-800`

### Editor Styling
- Split pane: left textarea with monospace font, right preview with article styles
- Textarea: `bg-stone-50`, `font-mono`, `text-sm`, generous line-height
- Preview: styled Markdown output matching public post page
- Divider: 1px stone-200 border, subtle resize handle

---

## 7. RSS Feed

Route: `GET /api/[username]/feed.xml`

Generate valid RSS 2.0 XML:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>{name}'s DevLog</title>
    <link>{APP_URL}/{username}</link>
    <description>Posts by {name}</description>
    <atom:link href="{APP_URL}/api/{username}/feed.xml" rel="self" type="application/rss+xml"/>
    <item>
      <title>{post.title}</title>
      <link>{APP_URL}/{username}/{post.slug}</link>
      <description>{post.excerpt}</description>
      <pubDate>{RFC 822 date}</pubDate>
      <guid isPermaLink="true">{APP_URL}/{username}/{post.slug}</guid>
    </item>
    ...
  </channel>
</rss>
```

Return with `Content-Type: application/xml`.

---

## 8. SEO

On public post pages (`/[username]/[slug]`), generate:
- `<title>` — post title + " | DevLog"
- `<meta name="description">` — excerpt
- `<meta property="og:title">`, `og:description`, `og:image` (cover image URL or default)
- `<meta property="og:type" content="article">`
- `<meta property="article:published_time">`, `article:author`
- `<link rel="alternate" type="application/rss+xml">` pointing to author's feed

Use Next.js `generateMetadata()` for dynamic meta tags.

---

## 9. Project Structure

```
devlog/
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── public/
├── src/
│   ├── app/
│   │   ├── layout.tsx              # Root layout (fonts, providers)
│   │   ├── page.tsx                # Public feed
│   │   ├── globals.css
│   │   ├── login/page.tsx
│   │   ├── register/page.tsx
│   │   ├── dashboard/page.tsx      # Protected
│   │   ├── write/page.tsx          # Protected
│   │   ├── edit/[id]/page.tsx      # Protected
│   │   ├── [username]/page.tsx     # Public author profile
│   │   ├── [username]/[slug]/page.tsx  # Public post (SSR)
│   │   └── api/
│   │       ├── auth/
│   │       │   ├── register/route.ts
│   │       │   ├── login/route.ts
│   │       │   ├── logout/route.ts
│   │       │   └── me/route.ts
│   │       ├── posts/
│   │       │   ├── route.ts            # GET (my posts), POST (create)
│   │       │   ├── [id]/route.ts       # GET, PUT, DELETE
│   │       │   └── published/route.ts  # GET (public feed)
│   │       ├── tags/route.ts
│   │       └── [username]/feed.xml/route.ts  # RSS
│   ├── components/
│   │   ├── ui/                 # Button, Input, Card, Modal, Badge, Spinner
│   │   ├── auth/               # LoginForm, RegisterForm, AuthGuard
│   │   ├── blog/               # PostCard, PostList, AuthorCard, TagChip
│   │   ├── editor/             # MarkdownEditor, SplitPane, CoverUpload, TagInput
│   │   └── layout/             # Navbar, Footer
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   └── usePosts.ts
│   ├── lib/
│   │   ├── prisma.ts           # Singleton Prisma client
│   │   ├── auth.ts             # JWT helpers, getAuthUser()
│   │   ├── fonts.ts            # next/font config
│   │   └── utils.ts            # slugify, readingTime, formatDate
│   ├── schemas/
│   │   ├── auth.ts             # Zod: register, login
│   │   └── post.ts             # Zod: create/update post
│   └── types/
│       └── index.ts
├── .env.example
├── .gitignore
├── next.config.ts
├── tailwind.config.ts
├── postcss.config.mjs
├── tsconfig.json
└── package.json
```

---

## 10. Environment Variables

### `.env.example`
```
DATABASE_URL="file:./dev.db"
JWT_SECRET=your-secret-key-change-in-production
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## 11. Seed Data

Create `prisma/seed.ts` that:

1. Creates a demo user:
   - Name: "Jamie Rivera"
   - Email: `demo@devlog.dev`
   - Password: `demo1234`
   - Bio: "Full-stack developer writing about JavaScript, system design, and the craft of software engineering."
   - Slug: `jamie-rivera`

2. Creates 4 published posts with realistic Markdown content including code blocks:
   - "Understanding JavaScript Closures" — tagged: javascript, fundamentals
   - "Designing REST APIs That Don't Make People Cry" — tagged: api-design, backend
   - "My First Year as a Software Engineer" — tagged: career, reflections
   - "A Practical Guide to Database Indexing" — tagged: databases, performance

3. Creates 1 draft post:
   - "Why TypeScript Changed How I Think About Code" — tagged: typescript, opinion

Each post should have 300-500 words of realistic content with Markdown formatting (headings, code blocks, lists, bold/italic, links).

---

## 12. Configuration

### `next.config.ts`
```ts
const nextConfig = {
  // No special config needed for SQLite + monolith
};
export default nextConfig;
```

### `tailwind.config.ts`
Extend with:
- Custom colors: `accent`, `surface` aliases
- Font families: `serif` → Newsreader, `sans` → Inter
- Typography plugin styles for article content

### `.gitignore`
```
node_modules/
.next/
*.db
*.db-journal
.env
.env.local
.DS_Store
*.tsbuildinfo
next-env.d.ts
```

### `package.json` scripts
```json
{
  "dev": "next dev",
  "build": "npx prisma generate && next build",
  "start": "next start",
  "lint": "next lint",
  "db:push": "npx prisma db push",
  "db:seed": "npx tsx prisma/seed.ts",
  "db:studio": "npx prisma studio"
}
```

---

## 13. Final Checks

Before delivering, verify:
- [ ] `npm run db:push` creates the SQLite database
- [ ] `npm run db:seed` populates demo data
- [ ] `npm run dev` starts without errors
- [ ] Register a new user, write a post, publish it, view it publicly
- [ ] Markdown renders correctly with syntax highlighting
- [ ] RSS feed returns valid XML
- [ ] Public post pages have correct meta tags
- [ ] Mobile layout is usable
- [ ] All form fields have labels, focus states work, no accessibility warnings
