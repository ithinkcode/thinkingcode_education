# BugHive — Spring Boot Backend API

Generate a complete REST API for a lightweight issue tracker using **Spring Boot 3.2+ with Java 21**. The API handles authentication, workspace/project management, issue CRUD with Kanban status tracking, comments, activity logging, and real-time board updates via SSE.

---

## 1. Technology Stack

| Layer | Choice |
|-------|--------|
| Language | Java 21 |
| Framework | Spring Boot 3.2+ |
| Build | Maven |
| Database | PostgreSQL via Spring Data JPA + Hibernate |
| Migrations | Flyway |
| Auth | Spring Security + JWT (stateless, `jjwt` library) |
| Validation | Jakarta Bean Validation (`@Valid`, `@NotBlank`, etc.) |
| DTO Mapping | MapStruct |
| Real-time | SSE via `SseEmitter` |
| Password hashing | BCrypt (Spring Security built-in) |
| API docs | SpringDoc OpenAPI (Swagger UI at `/swagger-ui.html`) |
| Dev tools | Spring Boot DevTools, Lombok |
| Containerization | Docker + `docker-compose` for local dev |

### Constraints
- **No AI/LLM calls** — all features are deterministic
- **No external file storage** — users get avatar colors (hex), not images
- **Stateless** — JWT-only, no HTTP sessions (`SessionCreationPolicy.STATELESS`)
- **No Spring Session, no OAuth2** — custom JWT filter only
- **No Kotlin** — pure Java 21 (use records for DTOs where MapStruct supports them)

---

## 2. Database Schema (Flyway Migrations)

Place migrations in `src/main/resources/db/migration/`.

### `V1__init.sql`

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    avatar_color VARCHAR(7) NOT NULL DEFAULT '#7c3aed',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    owner_id UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TYPE member_role AS ENUM ('OWNER', 'ADMIN', 'MEMBER');

CREATE TABLE workspace_members (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    role member_role NOT NULL DEFAULT 'MEMBER',
    joined_at TIMESTAMP NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, workspace_id)
);

CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    key VARCHAR(10) NOT NULL,
    description TEXT,
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (workspace_id, key)
);

CREATE TYPE issue_status AS ENUM ('BACKLOG', 'TODO', 'IN_PROGRESS', 'IN_REVIEW', 'DONE');
CREATE TYPE issue_priority AS ENUM ('NONE', 'LOW', 'MEDIUM', 'HIGH', 'URGENT');

CREATE TABLE issues (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    number INT NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status issue_status NOT NULL DEFAULT 'BACKLOG',
    priority issue_priority NOT NULL DEFAULT 'NONE',
    assignee_id UUID REFERENCES users(id) ON DELETE SET NULL,
    reporter_id UUID NOT NULL REFERENCES users(id),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    labels JSONB NOT NULL DEFAULT '[]',
    position DOUBLE PRECISION NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, number)
);

CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    author_id UUID NOT NULL REFERENCES users(id),
    issue_id UUID NOT NULL REFERENCES issues(id) ON DELETE CASCADE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TYPE activity_type AS ENUM (
    'ISSUE_CREATED', 'STATUS_CHANGED', 'PRIORITY_CHANGED',
    'ASSIGNEE_CHANGED', 'LABELS_CHANGED', 'COMMENT_ADDED',
    'COMMENT_EDITED', 'COMMENT_DELETED', 'TITLE_CHANGED',
    'DESCRIPTION_CHANGED'
);

CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type activity_type NOT NULL,
    issue_id UUID NOT NULL REFERENCES issues(id) ON DELETE CASCADE,
    actor_id UUID NOT NULL REFERENCES users(id),
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_issues_project_status ON issues(project_id, status);
CREATE INDEX idx_issues_assignee ON issues(assignee_id);
CREATE INDEX idx_activities_issue ON activities(issue_id);
CREATE INDEX idx_comments_issue ON comments(issue_id);
CREATE INDEX idx_workspace_members_workspace ON workspace_members(workspace_id);
```

### Issue Number Auto-Increment
Issue numbers are per-project. On insert, compute `MAX(number) + 1` for the project. Use `@PrePersist` or a service-layer method with pessimistic locking to avoid race conditions.

### Position for Ordering
Issues within a status column are ordered by `position` (double). When moving:
- Between two issues: `position = (above.position + below.position) / 2`
- To top: `position = firstIssue.position - 1000`
- To bottom: `position = lastIssue.position + 1000`
- Initial position: `1000 * issueNumber`

---

## 3. JPA Entities

Map each table to a JPA entity. Use:
- `@Entity`, `@Table(name = "...")`
- `@Id` with `@GeneratedValue(strategy = GenerationType.UUID)`
- `@Enumerated(EnumType.STRING)` for enums
- `@Column(columnDefinition = "jsonb")` for JSON fields with a custom `JsonbConverter` (AttributeConverter that serializes `List<String>` / `Map<String,Object>` to JSON string)
- `@CreationTimestamp`, `@UpdateTimestamp` for timestamps
- Proper `@ManyToOne`, `@OneToMany`, `@ManyToMany` relationships with `FetchType.LAZY`
- `@EntityListeners(AuditingEntityListener.class)` for audit timestamps

### Composite Key for WorkspaceMember
Use `@EmbeddedId` with a `WorkspaceMemberId` class (`userId` + `workspaceId`).

---

## 4. Authentication & Security

### JWT Implementation
Use `io.jsonwebtoken:jjwt-api` + `jjwt-impl` + `jjwt-jackson`.

**JwtService:**
- `generateAccessToken(userId)` — expires in 15 minutes
- `generateRefreshToken(userId)` — expires in 7 days
- `validateToken(token)` — returns claims or throws
- `getUserIdFromToken(token)` — extracts subject

**JwtAuthenticationFilter** (extends `OncePerRequestFilter`):
1. Read `Authorization: Bearer <token>` header
2. Validate token, extract userId
3. Load user from DB
4. Set `UsernamePasswordAuthenticationToken` in `SecurityContextHolder`
5. Skip filter for public endpoints

### Spring Security Config
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) {
        return http
            .cors(cors -> cors.configurationSource(corsConfigSource()))
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/api/health", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

### CORS
Allow origins from `ALLOWED_ORIGINS` env var (comma-separated). Allow credentials, standard headers + `Authorization`.

### Password Rules
Minimum 6 characters. Hash with BCrypt (strength 12).

---

## 5. API Endpoints

### 5.1 Auth (`/api/auth`)

| Method | Path | Body | Response | Auth |
|--------|------|------|----------|------|
| POST | `/register` | `{ name, email, password }` | `{ data: { user, accessToken, refreshToken } }` | No |
| POST | `/login` | `{ email, password }` | `{ data: { user, accessToken, refreshToken } }` | No |
| POST | `/refresh` | `{ refreshToken }` | `{ data: { accessToken, refreshToken } }` | No |
| GET | `/me` | — | `{ data: User }` | Yes |

### 5.2 Workspaces (`/api/workspaces`)

| Method | Path | Body | Response | Auth |
|--------|------|------|----------|------|
| GET | `/` | — | `{ data: Workspace[] }` (user's workspaces) | Yes |
| POST | `/` | `{ name }` | `{ data: Workspace }` | Yes |
| GET | `/{id}` | — | `{ data: Workspace }` (with members) | Yes (member) |
| PUT | `/{id}` | `{ name }` | `{ data: Workspace }` | Yes (owner/admin) |
| DELETE | `/{id}` | — | `204` | Yes (owner) |
| POST | `/{id}/invite` | `{ email, role }` | `{ data: WorkspaceMember }` | Yes (owner/admin) |
| DELETE | `/{id}/members/{userId}` | — | `204` | Yes (owner/admin, not self-remove owner) |

Slug: auto-generated from name (kebab-case). Append random 4-char suffix if duplicate.

### 5.3 Projects (`/api/workspaces/{wId}/projects` + `/api/projects/{id}`)

| Method | Path | Body | Response | Auth |
|--------|------|------|----------|------|
| GET | `/api/workspaces/{wId}/projects` | — | `{ data: Project[] }` | Yes (member) |
| POST | `/api/workspaces/{wId}/projects` | `{ name, key, description? }` | `{ data: Project }` | Yes (owner/admin) |
| GET | `/api/projects/{id}` | — | `{ data: Project }` (with issue counts per status) | Yes (member) |
| PUT | `/api/projects/{id}` | `{ name, description? }` | `{ data: Project }` | Yes (owner/admin) |
| DELETE | `/api/projects/{id}` | — | `204` | Yes (owner) |

Project key: 2-5 uppercase letters, unique within workspace. Validated with regex `^[A-Z]{2,5}$`.

### 5.4 Issues (`/api/projects/{pId}/issues` + `/api/issues/{id}`)

| Method | Path | Body | Response | Auth |
|--------|------|------|----------|------|
| GET | `/api/projects/{pId}/issues` | Query: `?status=TODO,IN_PROGRESS&priority=HIGH&assignee={id}&search=text&sort=created_at&order=desc` | `{ data: Issue[] }` | Yes (member) |
| POST | `/api/projects/{pId}/issues` | `{ title, description?, status?, priority?, assigneeId?, labels? }` | `{ data: Issue }` | Yes (member) |
| GET | `/api/issues/{id}` | — | `{ data: Issue }` (with assignee, reporter, comment count) | Yes (member) |
| PUT | `/api/issues/{id}` | Full update body | `{ data: Issue }` | Yes (member) |
| DELETE | `/api/issues/{id}` | — | `204` | Yes (reporter or owner/admin) |
| PATCH | `/api/issues/{id}/status` | `{ status, position }` | `{ data: Issue }` | Yes (member) |
| PATCH | `/api/issues/bulk` | `{ issueIds, status?, assigneeId?, priority? }` | `{ data: Issue[] }` | Yes (member) |

**Activity logging:** Every change to an issue (status, priority, assignee, labels, title, description) creates an Activity record. Include `oldValue` and `newValue` in metadata JSON.

**SSE broadcast:** After any issue mutation in a project, push an event to all SSE subscribers for that project.

### 5.5 Comments (`/api/issues/{iId}/comments` + `/api/comments/{id}`)

| Method | Path | Body | Response | Auth |
|--------|------|------|----------|------|
| GET | `/api/issues/{iId}/comments` | — | `{ data: Comment[] }` (with author) | Yes (member) |
| POST | `/api/issues/{iId}/comments` | `{ content }` | `{ data: Comment }` | Yes (member) |
| PUT | `/api/comments/{id}` | `{ content }` | `{ data: Comment }` | Yes (author only) |
| DELETE | `/api/comments/{id}` | — | `204` | Yes (author or owner/admin) |

Comment creation also creates a `COMMENT_ADDED` activity.

### 5.6 Activity (`/api/issues/{id}/activity`)

| Method | Path | Response | Auth |
|--------|------|----------|------|
| GET | `/api/issues/{id}/activity` | `{ data: Activity[] }` (with actor, chronological) | Yes (member) |

### 5.7 Real-Time SSE (`/api/projects/{id}/events`)

| Method | Path | Response | Auth |
|--------|------|----------|------|
| GET | `/api/projects/{id}/events` | `text/event-stream` | Yes (member, token as query param) |

**Event types sent over SSE:**
```
event: issue_created
data: { "issue": {...} }

event: issue_updated
data: { "issue": {...}, "changes": ["status", "priority"] }

event: issue_deleted
data: { "issueId": "..." }

event: comment_added
data: { "comment": {...}, "issueId": "..." }
```

**Implementation:**
- Use `SseEmitter` with 30-minute timeout
- Store emitters in a `ConcurrentHashMap<UUID, List<SseEmitter>>` keyed by project ID
- Clean up dead emitters on error/completion
- Send periodic heartbeat (`:ping` comment every 30s) to keep connections alive
- Auth: pass JWT as `?token=` query parameter (SSE doesn't support custom headers)

### 5.8 Health (`/api/health`)

Returns `{ "status": "ok", "timestamp": "..." }`. No auth required.

---

## 6. Package Structure

```
src/main/java/com/bughive/
├── BughiveApplication.java
├── config/
│   ├── SecurityConfig.java
│   ├── CorsConfig.java
│   ├── JacksonConfig.java
│   └── OpenApiConfig.java
├── security/
│   ├── JwtService.java
│   ├── JwtAuthenticationFilter.java
│   └── UserPrincipal.java
├── entity/
│   ├── User.java
│   ├── Workspace.java
│   ├── WorkspaceMember.java
│   ├── WorkspaceMemberId.java     # Composite key
│   ├── Project.java
│   ├── Issue.java
│   ├── Comment.java
│   ├── Activity.java
│   └── enums/
│       ├── MemberRole.java
│       ├── IssueStatus.java
│       ├── IssuePriority.java
│       └── ActivityType.java
├── repository/
│   ├── UserRepository.java
│   ├── WorkspaceRepository.java
│   ├── WorkspaceMemberRepository.java
│   ├── ProjectRepository.java
│   ├── IssueRepository.java
│   ├── CommentRepository.java
│   └── ActivityRepository.java
├── dto/
│   ├── request/
│   │   ├── RegisterRequest.java
│   │   ├── LoginRequest.java
│   │   ├── CreateWorkspaceRequest.java
│   │   ├── InviteMemberRequest.java
│   │   ├── CreateProjectRequest.java
│   │   ├── CreateIssueRequest.java
│   │   ├── UpdateIssueRequest.java
│   │   ├── UpdateStatusRequest.java
│   │   ├── BulkUpdateRequest.java
│   │   └── CreateCommentRequest.java
│   └── response/
│       ├── AuthResponse.java
│       ├── UserResponse.java
│       ├── WorkspaceResponse.java
│       ├── ProjectResponse.java
│       ├── IssueResponse.java
│       ├── CommentResponse.java
│       ├── ActivityResponse.java
│       └── ApiResponse.java        # Generic { data: T } wrapper
├── mapper/
│   ├── UserMapper.java             # MapStruct
│   ├── WorkspaceMapper.java
│   ├── ProjectMapper.java
│   ├── IssueMapper.java
│   ├── CommentMapper.java
│   └── ActivityMapper.java
├── service/
│   ├── AuthService.java
│   ├── WorkspaceService.java
│   ├── ProjectService.java
│   ├── IssueService.java
│   ├── CommentService.java
│   ├── ActivityService.java
│   └── SseService.java            # Manages emitters + broadcasts
├── controller/
│   ├── AuthController.java
│   ├── WorkspaceController.java
│   ├── ProjectController.java
│   ├── IssueController.java
│   ├── CommentController.java
│   ├── ActivityController.java
│   ├── SseController.java
│   └── HealthController.java
├── exception/
│   ├── GlobalExceptionHandler.java  # @RestControllerAdvice
│   ├── ResourceNotFoundException.java
│   ├── UnauthorizedException.java
│   ├── ForbiddenException.java
│   └── ConflictException.java
└── util/
    └── SlugUtil.java

src/main/resources/
├── application.yml
├── application-dev.yml
├── application-prod.yml
└── db/migration/
    ├── V1__init.sql
    └── V2__seed_data.sql
```

---

## 7. Configuration

### `application.yml`
```yaml
spring:
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5432/bughive}
    username: ${SPRING_DATASOURCE_USERNAME:postgres}
    password: ${SPRING_DATASOURCE_PASSWORD:postgres}
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
    open-in-view: false
  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: ${PORT:8080}

app:
  jwt:
    secret: ${JWT_SECRET:dev-secret-key}
    refresh-secret: ${JWT_REFRESH_SECRET:dev-refresh-secret}
    expiration-ms: ${JWT_EXPIRATION_MS:900000}
    refresh-expiration-ms: ${JWT_REFRESH_EXPIRATION_MS:604800000}
  cors:
    allowed-origins: ${ALLOWED_ORIGINS:http://localhost:3000}
```

### `application-dev.yml`
```yaml
spring:
  jpa:
    show-sql: true
logging:
  level:
    com.bughive: DEBUG
    org.springframework.security: DEBUG
```

### `pom.xml` key dependencies
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
</parent>

<properties>
    <java.version>21</java.version>
    <mapstruct.version>1.5.5.Final</mapstruct.version>
</properties>

<dependencies>
    <!-- Core -->
    <dependency>spring-boot-starter-web</dependency>
    <dependency>spring-boot-starter-data-jpa</dependency>
    <dependency>spring-boot-starter-security</dependency>
    <dependency>spring-boot-starter-validation</dependency>

    <!-- Database -->
    <dependency>postgresql</dependency>
    <dependency>flyway-core</dependency>

    <!-- JWT -->
    <dependency>io.jsonwebtoken:jjwt-api:0.12.5</dependency>
    <dependency>io.jsonwebtoken:jjwt-impl:0.12.5 (runtime)</dependency>
    <dependency>io.jsonwebtoken:jjwt-jackson:0.12.5 (runtime)</dependency>

    <!-- Mapping -->
    <dependency>org.mapstruct:mapstruct:${mapstruct.version}</dependency>

    <!-- Docs -->
    <dependency>org.springdoc:springdoc-openapi-starter-webmvc-ui:2.5.0</dependency>

    <!-- Dev -->
    <dependency>org.projectlombok:lombok (optional)</dependency>
    <dependency>spring-boot-devtools (runtime, optional)</dependency>

    <!-- Test -->
    <dependency>spring-boot-starter-test</dependency>
    <dependency>spring-security-test</dependency>
</dependencies>
```

Configure `maven-compiler-plugin` with MapStruct + Lombok annotation processors.

---

## 8. Error Handling

`@RestControllerAdvice` GlobalExceptionHandler:

| Exception | HTTP Status | Response |
|-----------|-------------|----------|
| `MethodArgumentNotValidException` | 400 | `{ error: "Validation failed", details: [{field, message}] }` |
| `ResourceNotFoundException` | 404 | `{ error: "Resource not found" }` |
| `UnauthorizedException` | 401 | `{ error: "Unauthorized" }` |
| `ForbiddenException` | 403 | `{ error: "Forbidden" }` |
| `ConflictException` | 409 | `{ error: "Conflict: ..." }` |
| `DataIntegrityViolationException` | 409 | `{ error: "Duplicate entry" }` |
| `Exception` (catch-all) | 500 | `{ error: "Internal server error" }` (log full stack trace) |

---

## 9. Authorization Rules

Implement authorization checks in service layer (not controllers):

- **Workspace access**: user must be a member of the workspace
- **Workspace admin actions** (update, invite, remove member): role must be `OWNER` or `ADMIN`
- **Workspace delete**: `OWNER` only
- **Project access**: user must be member of the project's workspace
- **Project admin actions** (create, update): `OWNER` or `ADMIN`
- **Project delete**: `OWNER` only
- **Issue CRUD**: any workspace member can create/read/update issues
- **Issue delete**: only reporter or workspace `OWNER`/`ADMIN`
- **Comment edit**: author only
- **Comment delete**: author or workspace `OWNER`/`ADMIN`
- **Owner cannot be removed** from workspace members

Use a helper method: `assertWorkspaceMember(workspaceId, userId)` and `assertWorkspaceRole(workspaceId, userId, minRole)`.

---

## 10. Docker & Deployment

### `Dockerfile`
```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/bughive-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build the JAR in CI or locally before Docker build:
```bash
./mvnw clean package -DskipTests
```

### `docker-compose.yml` (local dev)
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: bughive
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### `railway.toml`
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
internalPort = 8080
```

---

## 11. Seed Data (`V2__seed_data.sql`)

Insert via Flyway migration:

1. **Users** (passwords are BCrypt hashes of `demo1234`):
   - Alice Johnson (`alice@bughive.dev`) — avatar color `#7c3aed`
   - Bob Smith (`bob@bughive.dev`) — avatar color `#2563eb`

2. **Workspace**: "Acme Engineering" (slug: `acme-engineering`, owner: Alice)
   - Members: Alice (OWNER), Bob (MEMBER)

3. **Projects**:
   - "Platform" (key: `PLT`) — "Core platform services and APIs"
   - "Mobile App" (key: `MOB`) — "iOS and Android mobile application"

4. **Issues** (~15 total across both projects, all statuses):
   - PLT-1: "Set up CI/CD pipeline" — DONE, HIGH, assigned to Alice
   - PLT-2: "Implement user authentication" — DONE, HIGH, assigned to Alice
   - PLT-3: "Design database schema" — IN_REVIEW, MEDIUM, assigned to Bob
   - PLT-4: "Add rate limiting to API" — IN_PROGRESS, MEDIUM, assigned to Alice
   - PLT-5: "Write API documentation" — TODO, LOW, assigned to Bob
   - PLT-6: "Set up error monitoring" — TODO, MEDIUM, unassigned
   - PLT-7: "Optimize database queries" — BACKLOG, LOW, unassigned
   - PLT-8: "Add WebSocket support" — BACKLOG, NONE, unassigned
   - MOB-1: "Set up React Native project" — DONE, HIGH, assigned to Bob
   - MOB-2: "Implement login screen" — IN_PROGRESS, HIGH, assigned to Bob
   - MOB-3: "Design navigation structure" — IN_PROGRESS, MEDIUM, assigned to Alice
   - MOB-4: "Build dashboard view" — TODO, MEDIUM, unassigned
   - MOB-5: "Add push notifications" — BACKLOG, LOW, unassigned
   - MOB-6: "Implement offline mode" — BACKLOG, NONE, unassigned
   - MOB-7: "Write unit tests" — TODO, HIGH, assigned to Bob

5. **Comments** (3-4 on some issues):
   - PLT-3: Alice: "Looking good, just needs index on the user_email column"
   - PLT-4: Bob: "Should we use a token bucket or sliding window approach?"
   - PLT-4: Alice: "Token bucket — simpler and works well for our scale"
   - MOB-2: Alice: "Make sure to handle biometric auth on supported devices"

6. **Activities** for some of the issues showing status changes and assignments.

---

## 12. Final Checks

Before delivering, verify:
- [ ] `docker compose up -d` starts PostgreSQL
- [ ] `./mvnw spring-boot:run` starts without errors
- [ ] Flyway runs migrations and seed data on startup
- [ ] Register, login, get tokens, access protected endpoints
- [ ] Create workspace, invite member, create project, create issues
- [ ] Move issue status via PATCH, verify activity is logged
- [ ] SSE endpoint streams events when issues change
- [ ] Swagger UI loads at `/swagger-ui.html`
- [ ] CORS allows frontend origin
- [ ] Invalid requests return proper 400/401/403/404 responses
