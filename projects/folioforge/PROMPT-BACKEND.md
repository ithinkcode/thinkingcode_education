# Claude Code Prompt — Portfolio Platform Backend (Railway Deployment)

## Context

You are building the **backend API** for **FolioForge** — a portfolio generation platform used in a live education seminar. Students and graduates log in, answer a guided set of questions about themselves (or paste content from an existing resume), and the system generates a stunning, shareable portfolio page. This backend handles authentication, portfolio data persistence, PDF generation, and serves as the API layer consumed by a Next.js frontend deployed on Vercel.

**This backend will be deployed on Railway.** It must be fully containerized with a production-ready Dockerfile, a `railway.toml` config, and environment variable scaffolding for Railway's provisioned PostgreSQL.

---

## Technology Stack (Mandatory)

- **Runtime:** Node.js 20+ with TypeScript
- **Framework:** Express.js with strict typed middleware
- **Database:** PostgreSQL via Prisma ORM (Railway provides PostgreSQL — use `DATABASE_URL` env var)
- **Authentication:** JWT-based (access + refresh token pattern). Use `bcryptjs` for password hashing, `jsonwebtoken` for token signing.
- **PDF Generation:** Puppeteer (headless Chromium) rendering a self-contained HTML template into a downloadable PDF. The PDF must look professionally designed — not a plain text dump.
- **File Storage:** Generated PDFs stored temporarily on disk (Railway ephemeral storage) with signed URL-style download endpoints. For profile photos, accept base64 uploads and store as binary in PostgreSQL (or a `bytea` column) to avoid needing external object storage.
- **Validation:** Zod schemas for all request bodies
- **CORS:** Configurable origins via `ALLOWED_ORIGINS` env var (comma-separated), defaulting to `http://localhost:3000`

---

## Database Schema (Prisma)

Design and implement the following models:

### User
```
id            String   @id @default(uuid())
email         String   @unique
passwordHash  String
firstName     String
lastName      String
createdAt     DateTime @default(now())
updatedAt     DateTime @updatedAt
portfolio     Portfolio?
```

### Portfolio
```
id              String   @id @default(uuid())
userId          String   @unique
slug            String   @unique        // URL-safe, auto-generated from name, editable
headline        String                  // e.g. "Full-Stack Engineer & Systems Thinker"
summary         String   @db.Text       // 2-3 paragraph professional summary
photoBase64     String?  @db.Text       // profile photo as base64
skills          Json                    // Array of { name, proficiency, category }
experience      Json                    // Array of { company, role, startDate, endDate, highlights[] }
education       Json                    // Array of { institution, degree, field, year, achievements[] }
projects        Json                    // Array of { title, description, techStack[], liveUrl?, repoUrl?, imageBase64? }
certifications  Json                    // Array of { name, issuer, year, url? }
languages       Json                    // Array of { language, proficiency }
contact         Json                    // { email?, linkedin?, github?, twitter?, website? }
theme           String   @default("obsidian")  // theme preference for rendering
isPublished     Boolean  @default(false)
publishedAt     DateTime?
createdAt       DateTime @default(now())
updatedAt       DateTime @updatedAt
user            User     @relation(fields: [userId], references: [id], onDelete: Cascade)
```

Run `npx prisma generate` and `npx prisma db push` as part of the startup flow. Include a `prisma/seed.ts` with one demo portfolio so the frontend team has data to work with.

---

## API Endpoints

### Auth
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/register` | Create account. Returns access + refresh tokens. |
| POST | `/api/auth/login` | Email + password login. Returns tokens. |
| POST | `/api/auth/refresh` | Exchange refresh token for new access token. |
| GET  | `/api/auth/me` | Return current user profile (protected). |

### Portfolio (Protected — requires valid JWT)
| Method | Path | Description |
|--------|------|-------------|
| GET    | `/api/portfolio` | Get current user's portfolio (or 404 if none). |
| POST   | `/api/portfolio` | Create portfolio from onboarding answers. |
| PUT    | `/api/portfolio` | Update full portfolio. |
| PATCH  | `/api/portfolio` | Partial update (individual sections). |
| POST   | `/api/portfolio/publish` | Set `isPublished = true`, set `publishedAt`. |
| POST   | `/api/portfolio/unpublish` | Set `isPublished = false`. |
| DELETE | `/api/portfolio` | Delete portfolio entirely. |

### Public Portfolio (No auth required)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/public/portfolio/:slug` | Return published portfolio by slug. Returns 404 if not published. |

### PDF Generation (Protected)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio/pdf` | Generate and return PDF of current user's portfolio. Stream the PDF as `application/pdf` with `Content-Disposition: attachment`. |

### Resume Parsing (Protected)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/portfolio/parse-resume` | Accept `{ resumeText: string }` — plain text pasted from a resume. Use structured extraction logic (regex + heuristic patterns) to parse it into the portfolio JSON structure. Return the parsed data for the frontend to review/edit before saving. This does NOT need AI — use pattern matching for section headers like "Experience", "Education", "Skills", etc. |

### Health
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Return `{ status: "ok", timestamp, version }`. Used by Railway health checks. |

---

## PDF Generation — Design Requirements

The PDF is generated by Puppeteer rendering a **self-contained HTML string** (inline CSS, no external assets) into an A4 PDF. This is critical — the PDF must look like a beautifully designed document, not a browser print.

Design specifications for the PDF template:
- **Layout:** Two-column design. Left sidebar (30% width) with dark background containing photo, contact info, skills, and languages. Main content area (70%) with experience, education, projects, certifications.
- **Typography:** Use embedded Google Fonts via `@import` in the HTML `<style>` tag — use `Playfair Display` for headings and `Source Sans Pro` for body text.
- **Color:** Support multiple themes. Default "obsidian" theme uses deep charcoal (#1a1a2e) sidebar with gold (#e6b54a) accents. Include at least 3 themes: `obsidian`, `arctic` (white/ice-blue), `ember` (warm burgundy/cream).
- **Visual details:** Skill bars with animated-style fills, timeline dots for experience, subtle section dividers, the person's initials as a watermark in the background.
- **Page handling:** Use Puppeteer's `format: 'A4'`, `printBackground: true`, and proper page-break CSS to avoid cutting content mid-section.

---

## Middleware & Error Handling

- `authMiddleware` — extracts and validates JWT from `Authorization: Bearer <token>` header. Attaches `req.user = { id, email }`.
- `errorHandler` — global error handler that catches Zod validation errors, Prisma errors, JWT errors, and returns structured JSON: `{ error: string, details?: any, statusCode: number }`.
- `rateLimiter` — basic rate limiting (100 requests/min per IP) using `express-rate-limit`.
- Request logging middleware using `morgan` in `combined` format.

---

## Project Structure

```
backend/
├── src/
│   ├── index.ts                 # Express app bootstrap, middleware registration
│   ├── config/
│   │   └── env.ts               # Typed environment variable loader with defaults
│   ├── middleware/
│   │   ├── auth.ts
│   │   ├── errorHandler.ts
│   │   ├── rateLimiter.ts
│   │   └── validate.ts          # Zod validation middleware factory
│   ├── routes/
│   │   ├── auth.routes.ts
│   │   ├── portfolio.routes.ts
│   │   └── public.routes.ts
│   ├── controllers/
│   │   ├── auth.controller.ts
│   │   ├── portfolio.controller.ts
│   │   └── public.controller.ts
│   ├── services/
│   │   ├── auth.service.ts
│   │   ├── portfolio.service.ts
│   │   ├── pdf.service.ts       # Puppeteer PDF generation
│   │   └── resumeParser.service.ts
│   ├── schemas/
│   │   ├── auth.schema.ts       # Zod schemas for auth endpoints
│   │   └── portfolio.schema.ts  # Zod schemas for portfolio endpoints
│   ├── templates/
│   │   └── portfolioPdf.ts      # HTML template function for PDF rendering
│   └── utils/
│       ├── slugify.ts
│       └── tokens.ts            # JWT sign/verify helpers
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── Dockerfile
├── railway.toml
├── docker-compose.yml           # Local dev with PostgreSQL container
├── .env.example
├── tsconfig.json
├── package.json
└── README.md
```

---

## Dockerfile (Production)

```dockerfile
FROM node:20-slim

# Puppeteer dependencies
RUN apt-get update && apt-get install -y \
    chromium \
    fonts-liberation \
    libappindicator3-1 \
    libasound2 \
    libatk-bridge2.0-0 \
    libatk1.0-0 \
    libcups2 \
    libdbus-1-3 \
    libgdk-pixbuf2.0-0 \
    libnspr4 \
    libnss3 \
    libx11-xcb1 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    xdg-utils \
    --no-install-recommends && \
    rm -rf /var/lib/apt/lists/*

ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npx prisma generate
RUN npm run build

EXPOSE 3001
CMD ["npm", "start"]
```

---

## railway.toml

```toml
[build]
builder = "dockerfile"
dockerfilePath = "Dockerfile"

[deploy]
healthcheckPath = "/api/health"
healthcheckTimeout = 300
restartPolicyType = "on_failure"
restartPolicyMaxRetries = 3

[service]
internalPort = 3001
```

---

## docker-compose.yml (Local Development)

Provide a `docker-compose.yml` with:
- `postgres` service (PostgreSQL 16, mapped to port 5432, with volume persistence)
- `api` service (builds from Dockerfile, maps port 3001, depends on postgres, mounts source for hot-reload)

---

## Environment Variables (.env.example)

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/folioforge
JWT_SECRET=your-secret-key-change-in-production
JWT_REFRESH_SECRET=your-refresh-secret-change-in-production
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d
PORT=3001
ALLOWED_ORIGINS=http://localhost:3000
NODE_ENV=development
FRONTEND_URL=http://localhost:3000
```

---

## Seed Data

The `prisma/seed.ts` should create:
1. A demo user: `demo@folioforge.dev` / `demo1234`
2. A fully populated portfolio for that user with realistic data for a fictional "Alex Chen" — a full-stack developer with 3 years experience, 2 projects, relevant skills, education from a CS program, etc. Make the content realistic and impressive — this will be shown during the seminar demo.

---

## Scripts (package.json)

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "db:push": "prisma db push",
    "db:seed": "tsx prisma/seed.ts",
    "db:studio": "prisma studio",
    "lint": "eslint src/",
    "typecheck": "tsc --noEmit"
  }
}
```

---

## Final Checks

After generating all code:

1. Run `npm install`
2. Run `npx tsc --noEmit` — fix all type errors
3. Verify Dockerfile builds: `docker build -t folioforge-api .`
4. Ensure all route handlers are properly typed and connected
5. Verify Prisma schema is valid: `npx prisma validate`
6. Initialize a git repository, create a meaningful `.gitignore` (include `node_modules`, `dist`, `.env`, `.prisma`), and make an initial commit with message: `feat: FolioForge API — portfolio platform backend`
7. Log a startup banner showing the port, environment, and available routes

---

## IMPORTANT CONSTRAINTS

- **No AI/LLM calls** in the backend. Resume parsing uses heuristic pattern matching only. This keeps the backend simple, fast, and free of API key dependencies.
- **No external file storage.** Everything stays in PostgreSQL or ephemeral disk.
- **Stateless design.** No sessions. JWT-only auth. Ready for horizontal scaling.
- All responses follow a consistent envelope: `{ data: T }` for success, `{ error: string, details?: any }` for failures.
- Use `helmet` for security headers.
- Use `compression` for response compression.
