# ThinkingCode Education

A collection of full-stack projects built during **ThinkingCode Education** seminars. Each project includes the prompts, specifications, and folder scaffolding used to generate production-grade applications with AI-assisted development.

## Structure

```
projects/
└── <project-name>/
    ├── CLAUDE.md             # Project overview & conventions for Claude Code
    ├── PROMPT.md             # Generation prompt (monolith projects)
    ├── PROMPT-BACKEND.md     # Backend generation prompt (split projects)
    ├── PROMPT-FRONTEND.md    # Frontend generation prompt (split projects)
    ├── backend/              # Backend folder scaffold
    └── frontend/             # Frontend folder scaffold
```

## Projects

| # | Project | Architecture | Backend | Frontend | Database |
|---|---------|-------------|---------|----------|----------|
| 1 | [FolioForge](projects/folioforge/) | Split services | Express.js + TypeScript | Next.js 14 | PostgreSQL |
| 2 | [DevLog](projects/devlog/) | Monolith | Next.js API routes | Next.js 14 | SQLite |
| 3 | [BugHive](projects/bughive/) | Split services | Spring Boot 3 + Java 21 | Next.js 14 | PostgreSQL |

### FolioForge
Portfolio generation platform — users complete a guided onboarding wizard (or paste resume text) and the system generates a shareable portfolio page with PDF download.

### DevLog
Developer blogging platform — users write Markdown posts with live preview and syntax highlighting, published on a personal blog with RSS feed support.

### BugHive
Lightweight issue tracker with Kanban board — teams create workspaces, organize issues into projects, drag-and-drop across status columns, and track activity with real-time updates.

## How to Use

1. Browse into a project folder.
2. Read the prompt files (`PROMPT.md`, `PROMPT-BACKEND.md`, `PROMPT-FRONTEND.md`) — these are the full specifications used to generate the application source code.
3. The directory structure shows the intended project layout.
4. Copy the `CLAUDE.md` and prompt files into a fresh directory.
5. Use the prompts with an AI coding assistant (e.g. Claude Code) to generate the complete application.

Each project deliberately uses a different architecture and backend stack, giving you exposure to multiple real-world patterns.

## Resources

Guides and references for writing prompts that produce high-quality applications:

| Resource | Description |
|----------|-------------|
| [Prompt Writing Guide](resources/prompt-writing-guide.md) | The complete guide — structure, specificity, constraints, iteration, and common pitfalls |
| [Design Spec Cheatsheet](resources/design-spec-cheatsheet.md) | How to specify colors, fonts, components, and layouts so the output looks designed, not generated |
| [Prompt Template](resources/prompt-template.md) | Fill-in-the-blank template to start your own project prompt from scratch |

## License

For educational purposes only.
