# Design Specification Cheatsheet

How to describe visual design in prompts so the AI produces something with a real identity — not the default "AI startup landing page" look.

---

## The Problem

Without design direction, AI produces the same app every time:
- White background, blue primary button
- Inter or system fonts
- Generic card layouts with rounded corners
- Stock gradient hero section
- Cookie-cutter SaaS aesthetic

This cheatsheet gives you the vocabulary and patterns to break out of that default.

---

## Step 1: Name Your Design Direction

Give the design a short, evocative name and 1-2 reference points. This anchors the entire visual system.

| Name | Reference | Vibe |
|------|-----------|------|
| "Editorial Luxury" | Stripe docs × luxury brand | Dark, typographic, gold accents |
| "Clean Editorial" | Medium × dev.to | Light, serif headings, content-first |
| "Functional Clarity" | Linear × GitHub Issues | Data-dense, sidebar nav, monospace details |
| "Warm Craft" | Notion × Mailchimp | Cream backgrounds, rounded, friendly |
| "Terminal Noir" | Vercel × hacker aesthetic | Black bg, green/cyan accents, monospace |
| "Soft Brutalist" | Gumroad × Poolsuite | Bold colors, hard shadows, chunky type |

### How to Write It

```markdown
### Design Direction — "Editorial Luxury"
Dark mode default. Typography-driven with dramatic size contrast.
Think Stripe's documentation crossed with a luxury watch brand.
Muted gold accents on near-black backgrounds. Generous whitespace.
Subtle grain/noise texture overlay. No flashy gradients.
```

---

## Step 2: Define the Complete Color System

### Template

```markdown
### Colors
Background:      #______  (page base)
Surface:         #______  (cards, modals, elevated elements)
Surface hover:   #______  (card hover state)
Border:          #______  (dividers, input borders)
Border subtle:   #______  (very faint separators)

Text primary:    #______  (headings, body text)
Text secondary:  #______  (labels, timestamps, helper text)
Text muted:      #______  (placeholders, disabled text)

Accent:          #______  (primary buttons, links, active states)
Accent hover:    #______  (darker/lighter accent for hover)
Accent light:    #______  (accent backgrounds — tags, badges)

Success:         #______
Error:           #______
Warning:         #______
```

### Example Palettes

**Dark + Gold (Editorial Luxury)**
```
Background:    #0a0a0f     Surface:      #12121a
Border:        #1e1e2a     Text primary: #f5f5f0
Text secondary:#8a8a8a     Accent:       #d4a853
Success:       #7c9a82     Error:        #e54d4d
```

**Light + Indigo (Clean Editorial)**
```
Background:    #fafaf9     Surface:      #ffffff
Border:        #e7e5e4     Text primary: #1c1917
Text secondary:#78716c     Accent:       #4f46e5
Success:       #16a34a     Error:        #dc2626
```

**Light + Violet (Functional Clarity)**
```
Background:    #fafafa     Surface:      #ffffff
Sidebar:       #1e1b2e     Text primary: #111827
Text secondary:#6b7280     Accent:       #7c3aed
Success:       #16a34a     Error:        #dc2626
```

**Cream + Terracotta (Warm Craft)**
```
Background:    #faf5ef     Surface:      #ffffff
Border:        #e8ddd0     Text primary: #2d2418
Text secondary:#8a7a6b     Accent:       #c46a4a
Success:       #5a8a62     Error:        #c44a4a
```

---

## Step 3: Choose a Font Pairing

### Rules
1. **Never use Inter alone** — it's the AI default and screams "generated"
2. **Pair a display font with a body font** — contrast creates hierarchy
3. **Specify weights explicitly** — AI will often only use 400 and 700
4. **Specify the loading method** — `next/font/google`, self-hosted, etc.

### Proven Pairings

| Headings | Body | Vibe | Best For |
|----------|------|------|----------|
| Playfair Display (serif) | Source Sans Pro (sans) | Elegant, editorial | Portfolios, blogs |
| Clash Display (sans) | Satoshi (sans) | Bold, modern | SaaS, dashboards |
| Newsreader (serif) | Inter (sans) | Readable, warm | Blogs, content platforms |
| Space Grotesk (sans) | DM Sans (sans) | Geometric, tech | Dev tools, analytics |
| Fraunces (serif) | Work Sans (sans) | Friendly, crafted | Creative tools, marketplaces |
| JetBrains Mono (mono) | Inter (sans) | Technical, precise | Dev tools, code platforms |
| Sora (sans) | Nunito Sans (sans) | Clean, approachable | Education, productivity |

### How to Specify

```markdown
### Typography
- Headings: Playfair Display, serif — weights 400, 600, 700
- Body: Source Sans Pro, sans — weights 400, 500, 600
- Code/monospace: JetBrains Mono — weight 400, 500
- Load via next/font/google
- DO NOT use default system fonts, Inter alone, or Roboto

Size scale:
- Hero heading: 3.5rem / 4rem
- Page heading: 2rem / 2.5rem
- Section heading: 1.5rem
- Body: 1rem (16px), article body: 1.125rem (18px)
- Small/caption: 0.875rem (14px)
- Tiny/label: 0.75rem (12px)
```

---

## Step 4: Define Component Patterns

The goal: define each component once, then the AI applies it consistently everywhere.

### Button System

```markdown
### Buttons
All: rounded-lg font-medium transition-colors focus-visible:ring-2 ring-offset-2

- **Primary**: bg-accent text-white hover:bg-accent-hover
  `bg-[#4f46e5] text-white px-5 py-2.5 rounded-lg font-medium hover:bg-[#4338ca]`
- **Secondary**: border border-border bg-surface hover:bg-surface-hover
  `border border-[#e7e5e4] bg-white px-5 py-2.5 rounded-lg font-medium hover:bg-[#f5f5f4]`
- **Ghost**: transparent hover:bg-surface-hover
  `px-3 py-2 rounded-lg hover:bg-[#f5f5f4] text-secondary`
- **Danger**: bg-error text-white hover:bg-error-dark
  `bg-[#dc2626] text-white px-5 py-2.5 rounded-lg font-medium hover:bg-[#b91c1c]`

Sizes: sm (px-3 py-1.5 text-sm), md (default), lg (px-6 py-3 text-lg)
```

### Card System

```markdown
### Cards
- **Default**: bg-surface rounded-xl border border-border p-6
  Hover: shadow-md transition-shadow
- **Interactive**: add cursor-pointer, hover:border-accent/30
- **Elevated**: shadow-lg border-0 (modals, dropdowns)
- **Compact**: p-4 rounded-lg (list items, table rows)
```

### Input System

```markdown
### Inputs
- `bg-surface border border-border rounded-lg px-4 py-2.5`
- Focus: `border-accent ring-1 ring-accent`
- Error: `border-error ring-1 ring-error`
- Label: `text-sm font-medium text-primary mb-1.5 block`
- Helper text: `text-sm text-secondary mt-1`
- Error message: `text-sm text-error mt-1`
```

### Status Indicators

```markdown
### Status Badges
Pill shape: rounded-full px-2.5 py-0.5 text-xs font-medium

- Published/Active/Done: bg-emerald-100 text-emerald-800
- Draft/Pending/Todo:    bg-amber-100 text-amber-800
- Error/Failed/Urgent:   bg-red-100 text-red-800
- Info/In Progress:      bg-blue-100 text-blue-800
- Neutral/Backlog:       bg-gray-100 text-gray-700
```

---

## Step 5: Layout Patterns

### Sidebar + Content (Dashboards, Admin, Tools)

```markdown
### Layout
- Sidebar: fixed left, w-60, full height, dark bg (#1e1b2e)
  - Logo/app name at top
  - Nav links with icons (Lucide), active state = lighter bg + left accent border
  - User avatar + name at bottom
- Content: ml-60, p-8, light bg
  - Max content width: 1200px, centered
  - Page heading + actions row at top
```

### Centered Content (Blogs, Landing, Auth)

```markdown
### Layout
- Navbar: fixed top, bg-white, border-b, h-16
  - Logo left, nav links center or right
- Content: max-w-4xl mx-auto px-6 pt-24
- Article content: max-w-2xl (prose width) for readability
- Footer: border-t, py-8, muted text
```

### Split View (Editors, Comparisons)

```markdown
### Layout
- Full viewport height (h-screen)
- Left panel: 50% width, scrollable
- Divider: 1px border, optionally draggable
- Right panel: 50% width, scrollable
- Toolbar: fixed top within the split view area
```

---

## Step 6: Animation & Motion

### Less Is More

```markdown
### Animations
- Use Framer Motion for page transitions and component mounting
- All animations MUST respect prefers-reduced-motion
- Duration: 150-300ms for UI feedback, 300-500ms for page transitions
- Easing: ease-out for enters, ease-in for exits
- NO: parallax, infinite animations, scroll-jacking, loading spinners longer than 200ms
- YES: fade in content on mount, scale buttons on hover, slide modals in from bottom
```

### Specific Animation Patterns

```markdown
- Page content: fade-in + slide-up (y: 10px → 0, opacity: 0 → 1, 300ms)
- Modals: backdrop fade-in (200ms), content scale-up (0.95→1) + fade-in (200ms)
- Toasts: slide-in from top-right (300ms), auto-dismiss after 4s
- Cards on hover: shadow transition (200ms ease)
- Buttons on hover: background color transition (150ms)
- Sidebar nav: no animation, instant state change
```

---

## Step 7: Texture & Depth

These are optional details that elevate the design from "clean" to "designed."

### Grain/Noise Overlay (Dark Themes)

```markdown
Add a subtle noise texture overlay on the page background:
- CSS: fixed position div covering viewport
- Background: SVG noise pattern or tiny repeating PNG
- Opacity: 3-5% (barely visible, adds warmth)
- Pointer-events: none
- Mix-blend-mode: overlay
```

### Gradient Accents

```markdown
Decorative gradient banner on profile/hero sections:
- Height: h-48 to h-56
- Gradient: from-accent/15 via-surface to-background
- Profile photo overlaps the bottom edge: -mt-20, border-4 border-background
```

### Subtle Shadows (Light Themes)

```markdown
Shadow scale (use consistently):
- sm: 0 1px 2px rgba(0,0,0,0.05)    — default cards
- md: 0 4px 6px rgba(0,0,0,0.07)    — hover cards, dropdowns
- lg: 0 10px 15px rgba(0,0,0,0.1)   — modals, elevated panels
- xl: 0 20px 25px rgba(0,0,0,0.12)  — full-page overlays
```

---

## Quick Checklist

Before finalizing your design spec, verify:

- [ ] Design direction has a name and 1-2 references
- [ ] All colors defined with hex values and usage context
- [ ] Font pairing specified with family, weights, and sizes
- [ ] Fonts explicitly NOT Inter/Roboto/system (unless intentional)
- [ ] Button system: primary, secondary, ghost, danger + sizes
- [ ] Card pattern with border, padding, shadow, hover state
- [ ] Input pattern with focus, error, label, helper text states
- [ ] Status badges/pills defined with colors per state
- [ ] Layout pattern chosen (sidebar, centered, split)
- [ ] Animation guidelines: what to animate, durations, easing
- [ ] Light/dark mode decision made explicit
- [ ] Mobile behavior described (not just "responsive")
