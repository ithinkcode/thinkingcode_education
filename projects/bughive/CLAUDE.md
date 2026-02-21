# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**BugHive** — a lightweight issue tracker with Kanban board. Teams create workspaces, organize issues into projects, drag-and-drop across status columns, and track activity. Split into two independently deployed services: a Spring Boot backend API and a Next.js frontend.

## Architecture

- **Backend (`backend/`):** Spring Boot 3+ REST API deployed on **Railway** via Docker. Handles auth, workspace/project/issue CRUD, activity logging, and real-time updates via SSE.
- **Frontend (`frontend/`):** Next.js 14+ App Router deployed on **Vercel**. Kanban board with drag-and-drop, list view, issue detail panels, and workspace management.
- Communication is REST over HTTPS + SSE for real-time board updates. JWT-based stateless auth (access + refresh tokens).

## Backend

### Tech Stack
- Java 21, Spring Boot 3.2+
- Spring Data JPA + Hibernate + PostgreSQL
- Spring Security with JWT (stateless, no sessions)
- SSE via `SseEmitter` for real-time board updates
- Maven build
- Flyway for database migrations
- Bean Validation (Jakarta) + custom validators
- MapStruct for DTO mapping
- Dockerized for Railway deployment

### Commands
```bash
./mvnw spring-boot:run                    # Run dev server
./mvnw clean package                      # Build JAR
./mvnw clean package -DskipTests          # Build without tests
./mvnw test                               # Run tests
java -jar target/bughive-0.0.1.jar        # Run built JAR
```

### Key API Routes
- `POST /api/auth/register|login|refresh`, `GET /api/auth/me`
- `GET|POST /api/workspaces`, `GET|PUT|DELETE /api/workspaces/{id}`
- `POST /api/workspaces/{id}/invite`, `DELETE /api/workspaces/{id}/members/{userId}`
- `GET|POST /api/workspaces/{wId}/projects`, `GET|PUT|DELETE /api/projects/{id}`
- `GET|POST /api/projects/{pId}/issues`, `GET|PUT|DELETE /api/issues/{id}`
- `PATCH /api/issues/{id}/status` — update status (kanban move)
- `PATCH /api/issues/bulk` — bulk update status/assignee/priority
- `GET|POST /api/issues/{id}/comments`, `PUT|DELETE /api/comments/{id}`
- `GET /api/issues/{id}/activity` — activity feed
- `GET /api/projects/{id}/events` — SSE stream for real-time board
- `GET /api/health` — Railway health check

### Response Format
- Success: `{ "data": T }`
- Error: `{ "error": "message", "details": [...], "statusCode": 400 }`

### Database
JPA entities: `User`, `Workspace`, `WorkspaceMember`, `Project`, `Issue`, `Comment`, `Activity`. Flyway migrations under `src/main/resources/db/migration/`.

### Seed Data
Demo workspace "Acme Engineering" with 2 projects. Two users: `alice@bughive.dev` / `demo1234` (owner), `bob@bughive.dev` / `demo1234` (member). ~15 issues across all statuses with comments and activity.

## Frontend

### Tech Stack
- Next.js 14+ App Router, TypeScript strict mode
- Tailwind CSS v3 with custom design tokens
- HTML5 Drag and Drop API (custom implementation, no library)
- React Hook Form + Zod, Zustand (board state + optimistic updates)
- Framer Motion (subtle transitions only)
- Lucide React icons, Sonner toasts
- Fonts: JetBrains Mono (issue IDs, labels) + Inter (body) via `next/font`

### Commands
```bash
npm run dev          # next dev
npm run build        # next build
npm run lint         # next lint
```

### Routes
- `/` — landing page
- `/login`, `/register`
- `/[workspace]` — workspace dashboard (projects, recent activity, stats)
- `/[workspace]/settings` — workspace settings, members
- `/[workspace]/[project]` — **Kanban board** (default view)
- `/[workspace]/[project]/list` — list/table view with filtering & sorting
- `/[workspace]/[project]/[issueNumber]` — issue detail panel

### Design Direction — "Functional Clarity"
Light mode with dark sidebar. Productivity-focused, data-dense, visually quiet. Think Linear meets GitHub Issues.

- Sidebar: `#1e1b2e` (dark violet-gray)
- Content background: `#fafafa`, cards: `#ffffff`
- Accent: violet `#7c3aed`, hover: `#6d28d9`
- Monospace `JetBrains Mono` for issue keys (PLT-42), timestamps
- Priority indicators: colored dots (urgent=red, high=orange, medium=yellow, low=blue, none=gray)
- Keyboard shortcuts: `c` new issue, `1-5` priority, `/` search

### API Client
Typed client at `lib/api.ts` using native `fetch`. Auto-attaches JWT, handles 401 redirect, transparent token refresh. Base URL from `NEXT_PUBLIC_API_URL`.

## Critical Constraints

- **No AI/LLM calls** — all features are deterministic
- **No external file storage** — avatar colors instead of images, markdown descriptions only
- **No component libraries** (no shadcn, MUI, Chakra) — all UI custom-built including drag-and-drop
- **No Axios** — native `fetch` with typed wrapper
- **Custom drag-and-drop** — HTML5 DnD API, no external DnD library
- **Stateless** — JWT-only auth, no server sessions
- **Optimistic updates** — board moves update UI instantly, revert on server error
- **Responsive** — desktop primary, board scrolls horizontally on mobile
- **Accessibility** — keyboard board navigation, ARIA roles, focus management

## Environment Variables

### Backend (`application.yml` / env vars)
```
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/bughive
SPRING_DATASOURCE_USERNAME=postgres
SPRING_DATASOURCE_PASSWORD=postgres
JWT_SECRET=your-secret-key-change-in-production
JWT_REFRESH_SECRET=your-refresh-secret-change-in-production
JWT_EXPIRATION_MS=900000
JWT_REFRESH_EXPIRATION_MS=604800000
ALLOWED_ORIGINS=http://localhost:3000
```

### Frontend (`.env.local.example`)
```
NEXT_PUBLIC_API_URL=http://localhost:8080
NEXT_PUBLIC_APP_URL=http://localhost:3000
```
