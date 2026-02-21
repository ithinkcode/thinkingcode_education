# BugHive — Next.js Frontend (Kanban Issue Tracker)

Generate a complete frontend for a lightweight issue tracker with Kanban board using **Next.js 14+ App Router**. The frontend consumes a Spring Boot REST API and features drag-and-drop board management, list views with filtering, issue detail panels, workspace/project management, and real-time updates via SSE.

---

## 1. Technology Stack

| Layer | Choice |
|-------|--------|
| Framework | Next.js 14+ App Router |
| Language | TypeScript strict mode |
| Styling | Tailwind CSS v3 with custom design tokens |
| Drag & Drop | Custom implementation using HTML5 Drag and Drop API |
| State | Zustand (board state, optimistic updates) |
| Forms | React Hook Form + Zod validation |
| Animations | Framer Motion (subtle transitions only) |
| Icons | Lucide React |
| Toasts | Sonner |
| Fonts | JetBrains Mono (issue IDs, code) + Inter (body) via `next/font/google` |

### Constraints
- **No component libraries** (no shadcn, MUI, Chakra) — every component hand-built
- **No Axios** — native `fetch` with typed API wrapper
- **No DnD libraries** (no `@dnd-kit`, `react-beautiful-dnd`, `@hello-pangea/dnd`) — custom HTML5 DnD
- **No AI/LLM calls**
- **Responsive** — desktop primary, board scrolls horizontally on tablet/mobile
- **Accessibility** — keyboard navigation for board columns, ARIA roles, focus management
- **Optimistic updates** — board moves update UI instantly, revert on server error

---

## 2. API Client

Create a typed API client at `lib/api.ts`:

```ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL;
```

**Features:**
- Generic `apiFetch<T>(path, options)` wrapper
- Auto-attaches `Authorization: Bearer <token>` from Zustand auth store
- On 401: attempt token refresh via `/api/auth/refresh`, retry original request once
- If refresh fails: clear tokens, redirect to `/login`
- All responses typed as `ApiResponse<T> = { data: T }` or `ApiError = { error: string }`
- Throw typed errors that components can catch

**SSE client:**
- `subscribeToProject(projectId, token, onEvent)` function
- Opens `EventSource` to `${API_BASE}/api/projects/${id}/events?token=${token}`
- Parses typed events: `issue_created`, `issue_updated`, `issue_deleted`, `comment_added`
- Returns cleanup function to close connection
- Auto-reconnect on disconnect with exponential backoff (max 30s)

---

## 3. Auth

### Zustand Auth Store
```ts
interface AuthState {
  user: User | null;
  accessToken: string | null;
  refreshToken: string | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  register: (name: string, email: string, password: string) => Promise<void>;
  logout: () => void;
  refreshAuth: () => Promise<boolean>;
  checkAuth: () => Promise<void>;
}
```

- Persist tokens in `localStorage`
- `checkAuth()` called on app mount — calls `/api/auth/me` to validate token
- On login/register: store tokens + user, redirect to first workspace or workspace creation

### Auth Guard
Layout component at `(protected)/layout.tsx`:
- Shows loading spinner while `isLoading`
- Redirects to `/login` if no user after auth check
- Renders children if authenticated

---

## 4. Pages & Routes

### 4.1 Landing Page (`/`)

Productivity-focused hero:
- Headline: "Track bugs. Ship faster." in bold sans-serif
- Tagline: "A lightweight issue tracker with Kanban boards, built for small teams that move fast."
- CTA: "Get Started" → `/register`
- Feature highlights (3 cards): Kanban Board, Real-time Updates, Team Collaboration
- Clean, minimal — no excessive animations

### 4.2 Login & Register (`/login`, `/register`)

- Centered card on light gray background
- Form fields: email + password (login), name + email + password (register)
- Inline Zod validation errors
- "Don't have an account? Sign up" / "Already have an account? Log in" links
- On success: redirect to first workspace dashboard or `/create-workspace`

### 4.3 Workspace Dashboard (`/[workspace]`) — Protected

**Layout:** persistent sidebar (dark) + main content area.

**Sidebar** (rendered in protected layout, visible on all workspace pages):
- Workspace name + dropdown to switch workspaces
- Navigation links:
  - Projects (list of projects in workspace, each clickable)
  - Members
  - Settings
- "New Project" button
- User avatar (colored circle with initials) + name at bottom, logout option

**Dashboard content:**
- Workspace name heading
- Stats row: total issues, open issues, done this week, members count
- Projects grid: card per project showing name, key, issue count by status (mini bar chart or colored dots)
- Recent activity feed (last 10 activities across all projects)

### 4.4 Workspace Settings (`/[workspace]/settings`) — Protected

- Workspace name (editable by owner/admin)
- Members list: name, email, role badge, remove button (not for owner)
- Invite form: email + role select (ADMIN/MEMBER)
- Danger zone: delete workspace (owner only, with confirmation)

### 4.5 Kanban Board (`/[workspace]/[project]`) — Protected, **Primary View**

This is the core feature of the app.

**Header:**
- Project name + key badge (e.g., "Platform" `PLT`)
- View toggle: Board | List
- Filter bar: status (multi-select), priority (multi-select), assignee (select), search text
- "New Issue" button (opens modal)

**Board Layout:**
- 5 columns: BACKLOG, TODO, IN_PROGRESS, IN_REVIEW, DONE
- Each column:
  - Header: status label + issue count
  - Scrollable card list
  - "+" button at bottom to quick-create issue in that status

**Issue Cards (in columns):**
- Issue key in monospace (`PLT-42`) — colored by priority
- Title (truncated to 2 lines)
- Labels as small colored pills
- Bottom row: priority dot + assignee avatar (colored circle with initials) or "Unassigned"
- Click card → navigate to issue detail

**Drag and Drop:**
- Drag issue cards between columns (changes status)
- Drag to reorder within a column (changes position)
- Visual feedback:
  - Dragging card: reduced opacity, slight scale
  - Drop target column: highlighted border/background
  - Drop position indicator: thin line between cards
- On drop:
  1. Optimistically update Zustand board state
  2. Send `PATCH /api/issues/{id}/status` with new status + position
  3. On error: revert state, show error toast

**Real-time SSE:**
- Subscribe to project events on mount
- When another user moves/creates/deletes an issue, update board state
- Show subtle toast: "Bob moved PLT-5 to In Progress"

**Keyboard Shortcuts:**
- `c` — open new issue modal
- `1-5` — set priority on focused issue (1=none, 5=urgent)
- `/` — focus search input
- `Escape` — close modals

### 4.6 List View (`/[workspace]/[project]/list`) — Protected

Table/list alternative to the board:
- Columns: Key, Title, Status, Priority, Assignee, Labels, Updated
- Sortable columns (click header to toggle asc/desc)
- Same filter bar as board view
- Row click → navigate to issue detail
- Bulk actions: select multiple (checkboxes) → change status, priority, or assignee
- Pagination or infinite scroll

### 4.7 Issue Detail (`/[workspace]/[project]/[issueNumber]`) — Protected

Can render as a full page or slide-over panel (animate from right):
- **Header**: issue key (`PLT-42`) + title (editable inline)
- **Sidebar (right)**: status dropdown, priority dropdown, assignee dropdown, labels editor
- **Main content**:
  - Description (Markdown rendered, click to edit in textarea)
  - Comments section:
    - Chronological list, each with author avatar + name + timestamp + content (rendered Markdown)
    - "Write a comment" textarea at bottom with Submit button
    - Edit/delete own comments (inline)
  - Activity timeline (below comments):
    - Chronological, showing: "Alice changed status from TODO to IN_PROGRESS", "Bob added a comment", etc.
    - Each entry: actor avatar + description + timestamp
- **Actions**: delete issue (reporter or admin, with confirmation)

### 4.8 New Issue Modal

Triggered from board "New Issue" button or `c` shortcut:
- Title input (required)
- Description textarea (Markdown, optional)
- Status select (defaults to column if created from "+" button, else BACKLOG)
- Priority select
- Assignee select (workspace members)
- Labels input (comma-separated, freeform)
- "Create Issue" button

---

## 5. Design Tokens & Styling

### Colors
```
Sidebar bg:         #1e1b2e (dark violet-gray)
Sidebar text:       #e2e0ea
Sidebar hover:      #2d2a3e
Sidebar active:     #3b3756

Content bg:         #fafafa
Surface:            #ffffff
Border:             #e5e7eb (gray-200)

Text primary:       #111827 (gray-900)
Text secondary:     #6b7280 (gray-500)

Accent:             #7c3aed (violet-600)
Accent hover:       #6d28d9 (violet-700)
Accent light:       #ede9fe (violet-100)

Priority urgent:    #ef4444 (red-500)
Priority high:      #f97316 (orange-500)
Priority medium:    #eab308 (yellow-500)
Priority low:       #3b82f6 (blue-500)
Priority none:      #9ca3af (gray-400)

Status backlog:     #9ca3af (gray)
Status todo:        #6b7280 (dark gray)
Status in_progress: #3b82f6 (blue)
Status in_review:   #a855f7 (purple)
Status done:        #22c55e (green)

Success:            #16a34a
Error:              #dc2626
Warning:            #d97706
```

### Typography
- Body: `Inter`, weights 400/500/600/700
- Monospace: `JetBrains Mono` for issue keys, timestamps, code — weight 500
- Sidebar nav: `Inter` 14px medium
- Issue key: `JetBrains Mono` 12px, uppercase
- Page headings: 24px/28px bold
- Card titles: 14px medium

### Component Patterns

**Cards (issue cards in board):**
- `bg-white rounded-lg border border-gray-200 p-3 shadow-sm`
- `hover:shadow-md transition-shadow cursor-grab`
- While dragging: `opacity-50 shadow-lg rotate-1`

**Columns (kanban):**
- `bg-gray-50 rounded-xl p-2 min-w-[280px] max-w-[320px]`
- Column header: `px-3 py-2 font-semibold text-sm text-gray-600 uppercase tracking-wide`
- Status dot before column name (colored per status)

**Sidebar:**
- Fixed left, full height, `w-60`
- Dark background with light text
- Active nav item: lighter background + left accent border (violet)
- Workspace switcher: dropdown at top with workspace initial avatar

**Buttons:**
- Primary: `bg-violet-600 hover:bg-violet-700 text-white rounded-lg px-4 py-2 font-medium`
- Secondary: `bg-white border border-gray-300 hover:bg-gray-50 rounded-lg`
- Ghost: `hover:bg-gray-100 rounded-lg`
- Danger: `bg-red-600 hover:bg-red-700 text-white rounded-lg`
- All: `focus-visible:ring-2 ring-violet-500 ring-offset-2 transition-colors`

**Inputs:**
- `rounded-lg border border-gray-300 px-3 py-2 focus:border-violet-500 focus:ring-1 focus:ring-violet-500`

**Dropdowns (status, priority, assignee):**
- Custom-built, not `<select>`
- Trigger: button showing current value with chevron icon
- Popover: `bg-white rounded-lg shadow-lg border p-1 z-50`
- Options: `px-3 py-2 rounded-md hover:bg-gray-100 cursor-pointer`
- Use `focus-trap` behavior for keyboard navigation

**Badges/pills:**
- Status: colored background (light) + text matching status color, `rounded-full px-2 py-0.5 text-xs font-medium`
- Priority: small colored dot (8px circle) + label text
- Labels: `rounded-full bg-gray-100 text-gray-700 px-2 py-0.5 text-xs`

**Avatar circles:**
- `w-7 h-7 rounded-full flex items-center justify-center text-white text-xs font-bold`
- Background: user's `avatarColor`
- Content: first letter of name, uppercase

**Modal:**
- Overlay: `bg-black/50 backdrop-blur-sm`
- Content: `bg-white rounded-xl shadow-2xl max-w-lg w-full mx-4 p-6`
- Framer Motion: fade in overlay, scale + fade content

---

## 6. Zustand Board Store

```ts
interface BoardState {
  columns: Record<IssueStatus, Issue[]>;
  isLoading: boolean;

  // Data fetching
  fetchBoard: (projectId: string) => Promise<void>;

  // Optimistic mutations
  moveIssue: (issueId: string, toStatus: IssueStatus, toPosition: number) => void;
  revertMove: (issueId: string, fromStatus: IssueStatus, fromPosition: number) => void;
  addIssue: (issue: Issue) => void;
  updateIssue: (issue: Issue) => void;
  removeIssue: (issueId: string) => void;

  // SSE integration
  handleSseEvent: (event: BoardEvent) => void;
}
```

**Optimistic update flow:**
1. User drops card → `moveIssue()` updates store immediately
2. API call fires in background
3. On success: no action needed (state already correct)
4. On error: `revertMove()` restores original position, show error toast

---

## 7. Drag and Drop Implementation

Use the native HTML5 Drag and Drop API:

### Draggable Issue Card
```tsx
<div
  draggable
  onDragStart={(e) => {
    e.dataTransfer.setData('text/plain', issue.id);
    e.dataTransfer.effectAllowed = 'move';
    // Set drag image, add dragging class
  }}
  onDragEnd={() => {
    // Remove dragging class
  }}
>
```

### Droppable Column
```tsx
<div
  onDragOver={(e) => {
    e.preventDefault();
    e.dataTransfer.dropEffect = 'move';
    // Calculate drop position from mouse Y
    // Show drop indicator line
  }}
  onDragLeave={() => {
    // Remove drop indicator
  }}
  onDrop={(e) => {
    e.preventDefault();
    const issueId = e.dataTransfer.getData('text/plain');
    // Calculate new position
    // Optimistic update via Zustand
    // API call
  }}
>
```

### Position Calculation
- Track mouse Y position relative to cards in the column
- Show a thin horizontal indicator line between cards at the drop position
- Calculate position value:
  - Between cards A and B: `(A.position + B.position) / 2`
  - Before first card: `firstCard.position - 1000`
  - After last card: `lastCard.position + 1000`
  - Empty column: `1000`

---

## 8. Project Structure

```
frontend/
├── public/
├── src/
│   ├── app/
│   │   ├── layout.tsx                 # Root layout (fonts, toaster)
│   │   ├── page.tsx                   # Landing page
│   │   ├── globals.css
│   │   ├── login/page.tsx
│   │   ├── register/page.tsx
│   │   └── (protected)/
│   │       ├── layout.tsx             # Auth guard + sidebar layout
│   │       ├── [workspace]/
│   │       │   ├── page.tsx           # Workspace dashboard
│   │       │   ├── settings/page.tsx
│   │       │   └── [project]/
│   │       │       ├── page.tsx       # Kanban board (default)
│   │       │       ├── list/page.tsx  # List/table view
│   │       │       └── [issueNumber]/page.tsx  # Issue detail
│   ├── components/
│   │   ├── ui/                # Button, Input, Select, Modal, Badge, Spinner, Dropdown, Avatar
│   │   ├── auth/              # LoginForm, RegisterForm
│   │   ├── layout/            # Sidebar, WorkspaceSwitcher, NavLink
│   │   ├── workspace/         # WorkspaceCard, MemberList, InviteForm, StatsRow
│   │   ├── board/             # KanbanBoard, BoardColumn, IssueCard, DropIndicator, NewIssueModal, FilterBar, ViewToggle
│   │   └── issue/             # IssueDetail, IssueDescription, CommentList, CommentForm, ActivityTimeline, StatusSelect, PrioritySelect, AssigneeSelect, LabelEditor
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useSse.ts          # SSE subscription hook
│   │   └── useKeyboardShortcuts.ts
│   ├── lib/
│   │   ├── api.ts             # Typed fetch client
│   │   ├── sse.ts             # SSE client
│   │   ├── fonts.ts           # next/font config
│   │   └── utils.ts           # formatDate, getInitials, getPriorityColor, getStatusColor
│   ├── stores/
│   │   ├── authStore.ts       # Zustand auth
│   │   └── boardStore.ts      # Zustand board state + optimistic updates
│   ├── schemas/
│   │   ├── auth.ts            # Zod: login, register
│   │   ├── issue.ts           # Zod: create/update issue
│   │   └── workspace.ts       # Zod: create workspace, invite
│   └── types/
│       └── index.ts           # All TypeScript interfaces
├── .env.local.example
├── .gitignore
├── next.config.ts
├── tailwind.config.ts
├── postcss.config.mjs
├── tsconfig.json
└── package.json
```

---

## 9. TypeScript Types

```ts
// Enums
type IssueStatus = 'BACKLOG' | 'TODO' | 'IN_PROGRESS' | 'IN_REVIEW' | 'DONE';
type IssuePriority = 'NONE' | 'LOW' | 'MEDIUM' | 'HIGH' | 'URGENT';
type MemberRole = 'OWNER' | 'ADMIN' | 'MEMBER';
type ActivityType = 'ISSUE_CREATED' | 'STATUS_CHANGED' | 'PRIORITY_CHANGED'
  | 'ASSIGNEE_CHANGED' | 'LABELS_CHANGED' | 'COMMENT_ADDED'
  | 'COMMENT_EDITED' | 'COMMENT_DELETED' | 'TITLE_CHANGED'
  | 'DESCRIPTION_CHANGED';

interface User {
  id: string;
  name: string;
  email: string;
  avatarColor: string;
  createdAt: string;
}

interface Workspace {
  id: string;
  name: string;
  slug: string;
  ownerId: string;
  members?: WorkspaceMember[];
  createdAt: string;
}

interface WorkspaceMember {
  userId: string;
  workspaceId: string;
  role: MemberRole;
  user: User;
  joinedAt: string;
}

interface Project {
  id: string;
  name: string;
  key: string;
  description?: string;
  workspaceId: string;
  issueCounts?: Record<IssueStatus, number>;
  createdAt: string;
}

interface Issue {
  id: string;
  number: number;
  title: string;
  description?: string;
  status: IssueStatus;
  priority: IssuePriority;
  assignee?: User;
  reporter: User;
  projectId: string;
  labels: string[];
  position: number;
  commentCount?: number;
  createdAt: string;
  updatedAt: string;
}

interface Comment {
  id: string;
  content: string;
  author: User;
  issueId: string;
  createdAt: string;
  updatedAt: string;
}

interface Activity {
  id: string;
  type: ActivityType;
  issueId: string;
  actor: User;
  metadata: Record<string, any>;
  createdAt: string;
}

// SSE Events
type BoardEvent =
  | { type: 'issue_created'; issue: Issue }
  | { type: 'issue_updated'; issue: Issue; changes: string[] }
  | { type: 'issue_deleted'; issueId: string }
  | { type: 'comment_added'; comment: Comment; issueId: string };
```

---

## 10. Environment & Configuration

### `.env.local.example`
```
NEXT_PUBLIC_API_URL=http://localhost:8080
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### `next.config.ts`
```ts
const nextConfig = {
  // Standard config, no special requirements
};
export default nextConfig;
```

### `tailwind.config.ts`
Extend with:
- Custom colors for priorities, statuses, sidebar
- Font families: `mono` → JetBrains Mono, `sans` → Inter
- Custom `min-w` and `max-w` for kanban columns

### `.gitignore`
```
node_modules/
.next/
.env
.env.local
.DS_Store
*.tsbuildinfo
next-env.d.ts
```

### `vercel.json`
```json
{
  "framework": "nextjs",
  "buildCommand": "npm run build"
}
```

---

## 11. Final Checks

Before delivering, verify:
- [ ] `npm run dev` starts without errors
- [ ] Login with demo credentials, see workspace dashboard
- [ ] Navigate to project board, see issues in correct columns
- [ ] Drag issue between columns — optimistic update works
- [ ] Drag to reorder within a column
- [ ] Open issue detail, edit title, change status/priority/assignee
- [ ] Add a comment, see it appear
- [ ] Activity timeline shows history
- [ ] List view sorts and filters correctly
- [ ] Bulk select + status change works
- [ ] Keyboard shortcuts (`c` to create, `/` to search) work
- [ ] Open two browser tabs — SSE shows real-time updates from other tab
- [ ] New issue modal validates required fields
- [ ] Mobile: board scrolls horizontally, sidebar collapses
- [ ] All interactive elements are keyboard-navigable
