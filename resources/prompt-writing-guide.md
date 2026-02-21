# The Prompt Engineering Guide for AI-Generated Applications

How to write prompts that produce production-grade, well-designed applications — not generic AI slop.

---

## The Core Principle

**Vague prompts produce vague apps.** The quality of AI-generated code is directly proportional to the specificity of your prompt. A prompt that says "make it look good" will produce something that looks like every other AI-generated app. A prompt that says `#d4a853 gold accent on #0a0a0f charcoal background, Playfair Display headings at 2.5rem` will produce something with an actual identity.

---

## 1. Structure Your Prompt Like a Blueprint

Every great generation prompt follows a predictable structure. Think of it as a technical specification, not a wish list.

### The Anatomy of a Complete Prompt

```
1.  Technology Stack         — What to build with
2.  Constraints              — What NOT to do (just as important)
3.  Database Schema          — The data model, fully defined
4.  Authentication           — Auth flow, token strategy, middleware
5.  API Endpoints            — Every route, method, body, response
6.  Pages & Layouts          — Every screen described in detail
7.  Design Tokens            — Colors, fonts, spacing, component patterns
8.  Project Structure        — Exact file tree
9.  Configuration            — Package scripts, env vars, config files
10. Seed Data                — Realistic demo data
11. Final Checks             — Verification checklist
```

**Why this order matters:** Each section builds on the previous one. The database schema informs the API design. The API design informs the page structure. The page structure informs the component design. Skip a section and the AI will guess — and AI guesses tend to be generic.

---

## 2. Be Specific Where It Matters

### Bad: Abstract descriptions
```
Use a modern color scheme with good contrast.
The design should feel professional and clean.
Add nice animations.
```

### Good: Concrete specifications
```
Background: #0a0a0f (near-black), Cards: #12121a (dark gray)
Accent: #d4a853 (muted gold), Secondary: #7c9a82 (sage green)
Text: #f5f5f0 primary, #8a8a8a secondary
Headings: Playfair Display serif 700, Body: Source Sans Pro 400
Animations: Framer Motion, all respect prefers-reduced-motion
```

### The Rule of Thumb
If you can picture it in your head but not on screen from your description alone, it's not specific enough. A designer reading your prompt should be able to create a mockup without asking you a single question.

---

## 3. Constraints Are Your Secret Weapon

Telling the AI what **not** to do is often more valuable than telling it what to do. Without constraints, AI will default to:
- Generic component libraries (shadcn, MUI)
- Default fonts (Inter, system fonts)
- Boilerplate color schemes (generic blue/white)
- Over-engineered abstractions
- Kitchen-sink features

### Essential Constraints to Always Define

```markdown
### Constraints
- **No component libraries** (no shadcn, MUI, Chakra) — all UI custom-built
- **No Axios** — use native fetch with typed wrapper
- **No AI/LLM calls** — all features deterministic
- **Responsive** — desktop primary, must work on mobile
- **Accessibility** — heading hierarchy, form labels, focus states
```

**Why "no component libraries"?** Two reasons:
1. Students learn more by building UI from scratch
2. The output has a distinctive look instead of the "shadcn starter template" aesthetic that plagues AI-generated apps

---

## 4. Design Specifications That Actually Work

This is where most prompts fail. Here's how to specify design that produces distinctive results.

### Give Your Design a Name and Reference Point

```markdown
### Design Direction — "Editorial Luxury"
Dark mode default. Think Stripe's documentation meets a luxury brand.
```

The name ("Editorial Luxury") gives the AI a coherent design philosophy to work from. The reference point ("Stripe meets luxury brand") anchors it in something real. Without this, you get "generic SaaS dashboard #47,000."

### Specify the Complete Color System

Don't just define primary/secondary. Define the full palette with where each color is used:

```markdown
### Colors
Background:     #0a0a0f (base)
Surface:        #12121a (cards, modals)
Border:         #1e1e2a (subtle dividers)
Text primary:   #f5f5f0 (headings, body)
Text secondary: #8a8a8a (labels, timestamps)
Accent:         #d4a853 (gold — CTAs, links, highlights)
Accent hover:   #c49a48 (darker gold)
Secondary:      #7c9a82 (sage — success states, tags)
Error:          #e54d4d
Warning:        #d4a853 (same as accent — intentional)
```

### Specify Font Pairing with Weights

```markdown
### Typography
- Headings: Playfair Display serif — weights 400, 600, 700
- Body: Source Sans Pro sans — weights 400, 500, 600
- Code: JetBrains Mono — weight 500
- Load via next/font/google — no Inter, no Roboto, no system fonts
```

**The "no Inter" rule:** Inter is the default AI font. If you don't explicitly specify something else AND ban defaults, you'll get Inter. Same for system fonts.

### Define Component Patterns with Actual CSS

```markdown
### Component Patterns
- **Cards**: bg-[#12121a] rounded-xl border border-[#1e1e2a] p-6 hover:border-[#d4a853]/30 transition
- **Buttons (primary)**: bg-[#d4a853] text-[#0a0a0f] rounded-lg px-6 py-2.5 font-semibold hover:bg-[#c49a48]
- **Inputs**: bg-[#12121a] border border-[#1e1e2a] rounded-lg px-4 py-2.5 text-[#f5f5f0] focus:border-[#d4a853] focus:ring-1 focus:ring-[#d4a853]
```

This looks pedantic, but it's the difference between an app that looks designed and one that looks generated.

---

## 5. Database Schema: Be the Architect

Don't say "create a database for blog posts." Write the actual schema.

### Bad
```
The app needs users and blog posts. Posts can have tags.
Users can save posts as drafts or publish them.
```

### Good
```prisma
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
```

**Why write the full schema?** Because data modeling is a design decision. If you leave it to the AI, you'll get a schema that works but isn't optimized for your access patterns. You'll also get inconsistent naming conventions and missing constraints.

---

## 6. API Endpoints: Use Tables, Not Prose

Tables are unambiguous. Prose is not.

### Bad
```
The API should let users create, read, update, and delete posts.
Users need to be authenticated for most operations.
There should be a public endpoint for viewing published posts.
```

### Good
```markdown
| Method | Route | Body | Response | Auth |
|--------|-------|------|----------|------|
| GET | `/api/posts` | — | `{ data: Post[] }` (own posts) | Yes |
| POST | `/api/posts` | `{ title, content, tags? }` | `{ data: Post }` | Yes |
| GET | `/api/posts/[id]` | — | `{ data: Post }` | Mixed* |
| PUT | `/api/posts/[id]` | `{ title, content, tags? }` | `{ data: Post }` | Yes (own) |
| DELETE | `/api/posts/[id]` | — | `204` | Yes (own) |
| GET | `/api/posts/published` | Query: `?tag=x&page=1` | `{ data: Post[] }` | No |

*Mixed: auth required for own drafts, public for published posts
```

---

## 7. Page Descriptions: Walk Through the Screen

Describe each page as if you're walking someone through a Figma mockup, top to bottom.

### Bad
```
The dashboard should show the user's posts with options to edit or delete them.
```

### Good
```markdown
### Dashboard (`/dashboard`) — Protected
- Stats row at top: 3 cards showing total posts, published count, total reading time
- Tabs below: "Published" | "Drafts" — pill-style toggle
- Post list (sorted by updatedAt desc):
  - Each row: title (link to edit), status badge (Published=green, Draft=amber),
    date, reading time, tag chips, action menu (edit, delete, view if published)
- Empty state (no posts): illustration placeholder, "Write your first post" CTA button
- Floating action button on mobile: "+" → /write
```

**The pattern:** Section → Content → Interaction → Edge cases (empty states, mobile)

---

## 8. Seed Data: Make It Real

AI-generated seed data is notoriously terrible — "Lorem ipsum" text, "Test User" names, "Sample Post" titles. Specify realistic data.

```markdown
### Seed Data
Create demo user "Jamie Rivera" (demo@devlog.dev / demo1234) with:

- "Understanding JavaScript Closures" — 400 words with code blocks,
  tagged: javascript, fundamentals
- "Designing REST APIs That Don't Make People Cry" — 350 words,
  tagged: api-design, backend
- "My First Year as a Software Engineer" — 500 words, personal tone,
  tagged: career, reflections

Each post: realistic Markdown with headings, code blocks, lists, bold/italic.
```

Good seed data matters because:
1. It's the first thing anyone sees when running the app
2. It tests that your rendering pipeline handles real content
3. It makes demos look professional

---

## 9. The Final Checks Section

End every prompt with a verification checklist. This does two things:
1. Tells the AI exactly what "done" looks like
2. Gives you a testing script to follow

```markdown
## Final Checks
Before delivering, verify:
- [ ] `npm run dev` starts without errors
- [ ] Register a new user, complete the full flow
- [ ] All CRUD operations work end to end
- [ ] Mobile layout is usable
- [ ] Form validation shows errors correctly
- [ ] Empty states render (no data = no crash)
- [ ] Auth redirects work (protected pages redirect to login)
```

---

## 10. Common Pitfalls

### Pitfall 1: Describing features instead of behavior
- Bad: "Add a search feature"
- Good: "Search input in the header, debounced 300ms, filters posts by title and content, shows results in a dropdown with post title + excerpt, Enter navigates to first result, Escape closes dropdown"

### Pitfall 2: Forgetting error states and edge cases
Always specify: What happens when the API fails? What does an empty list look like? What if the user uploads a 10MB image? What if the slug already exists?

### Pitfall 3: Not specifying the response format
If you don't define `{ data: T }` vs `{ result: T }` vs bare `T`, the AI will be inconsistent across endpoints. Pick a format and document it once.

### Pitfall 4: Leaving font choices to the AI
You will get Inter. Every time. Unless you explicitly choose something else AND tell it not to use defaults.

### Pitfall 5: Skipping the project structure
Without an explicit file tree, the AI will organize code however it wants. This leads to inconsistent patterns across files and makes the codebase harder to navigate.

### Pitfall 6: Not defining auth flow completely
"Add authentication" produces wildly different results. Specify: registration fields, login flow, token type (JWT/session), storage (cookies/localStorage/headers), expiration, refresh strategy, what happens on 401, and which routes are protected.

---

## 11. The Iteration Loop

Your first prompt won't produce a perfect app. Here's the iteration workflow:

1. **Generate** — Run the full prompt
2. **Run** — Start the app, click through every page
3. **Capture** — Screenshot issues, note bugs, identify gaps
4. **Refine** — Add specificity to the sections that produced poor results
5. **Regenerate** — Run the refined prompt (or patch specific sections)

Each iteration should make your prompt more specific, not longer. If a section is already producing good results, leave it alone. Focus your energy on the sections that need more detail.

---

## Quick Reference: Prompt Quality Checklist

Before running your prompt, verify:

- [ ] **Tech stack** is fully specified with versions
- [ ] **Constraints** section exists (what NOT to use)
- [ ] **Database schema** is written in actual schema syntax
- [ ] **Every API endpoint** is in a table with method, route, body, response, auth
- [ ] **Every page** is described top-to-bottom with layout details
- [ ] **Colors** are hex values, not descriptions ("warm" ≠ `#fafaf9`)
- [ ] **Fonts** are named explicitly with weights
- [ ] **Component patterns** include actual Tailwind/CSS classes
- [ ] **Project structure** is an explicit file tree
- [ ] **Seed data** uses realistic names, titles, and content
- [ ] **Final checks** section exists with verification steps
- [ ] **No vague language** — search for "nice", "clean", "modern", "good" and replace with specifics
