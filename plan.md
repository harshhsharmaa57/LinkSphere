# LinkSphere — Implementation Plan

> A community-driven platform to discover, save, and share the best resources on the internet.

---

## Phase 0: Project Setup ✅ (Partially Done)

### Backend (`linksphere-backend/`)
- [x] Folder structure created
- [x] `go mod init github.com/harshhsharmaa/linksphere-backend`
- [x] All Go dependencies installed (Gin, GORM, JWT, Colly, Redis, Cloudinary, etc.)
- [ ] Create `.env.example` with all required env vars
- [ ] Create `.gitignore`

### Frontend (`linksphere-frontend/`)
- [x] Folder structure created
- [ ] Initialize Vite with React + TypeScript template
- [ ] Install core deps: `axios`, `react-router-dom`, `@tanstack/react-query`
- [ ] Install Tailwind CSS + shadcn/ui
- [ ] Create `.env` with `VITE_API_URL=http://localhost:8080/api`
- [ ] Create `.gitignore`

---

## Phase 1: Backend Core Infrastructure

> **Goal:** Config loading, database connection, Redis init, and server boot.

### 1.1 — Config (`pkg/config/config.go`)
- Read all env vars into a `Config` struct using `godotenv` + `os.Getenv`
- Fields: `Port`, `DatabaseURL`, `RedisURL`, `JWTSecret`, `JWTExpiryHours`, `CloudinaryCloudName`, `CloudinaryAPIKey`, `CloudinaryAPISecret`, `FrontendURL`
- Validate required fields, panic on missing critical vars

### 1.2 — Database (`pkg/database/postgres.go`)
- Open GORM connection with `gorm.io/driver/postgres`
- Run `AutoMigrate` for all models: `User`, `Link`, `Vote`, `Save`, `Collection`
- Export a `ConnectDB(dsn string) *gorm.DB` function

### 1.3 — Redis (`pkg/cache/redis.go`)
- Parse `REDIS_URL`, create `go-redis` client
- Export `ConnectRedis(url string) *redis.Client`
- Add a `Ping()` health check

### 1.4 — Cloudinary (`pkg/cloudinary/upload.go`)
- Init Cloudinary client from env vars
- Export `UploadImage(file io.Reader) (string, error)` → returns `secure_url`

### 1.5 — SQL Migrations (`migrations/`)
| File | Table | Notes |
|------|-------|-------|
| `001_create_users.sql` | `users` | UUID PK, unique username/email, bcrypt password |
| `002_create_links.sql` | `links` | FK to users, category, tags array, indexes |
| `003_create_votes.sql` | `votes` | FK to users+links, value CHECK(-1,1), unique constraint |
| `004_create_saves.sql` | `saves` | FK to users+links, unique constraint |
| `005_create_collections.sql` | `collections` | FK to users, is_public default true |
| `006_create_collection_links.sql` | `collection_links` | Composite PK (collection_id, link_id) |

### 1.6 — Middleware
- **`internal/middleware/cors.go`** — Allow `FRONTEND_URL` origin, standard headers, credentials
- **`internal/middleware/logger.go`** — Gin request logger with method, path, status, latency
- **`internal/middleware/ratelimit.go`** — Redis sliding window, 60 req/min per IP

### 1.7 — Server Entrypoint (`cmd/server/main.go`)
- Load config → Connect DB → Connect Redis → Init Cloudinary
- Register all middleware (CORS, Logger, RateLimit)
- Register all route groups (auth, users, links, collections, search)
- Start Gin on `config.Port`

---

## Phase 2: Authentication System

> **Goal:** User registration, login, JWT tokens, and auth middleware.

### 2.1 — JWT Utilities (`internal/auth/jwt.go`)
- `GenerateToken(userID uuid.UUID) (string, error)` — HS256, expiry from config
- `ParseToken(tokenStr string) (uuid.UUID, error)` — validate and extract userID

### 2.2 — Auth Middleware (`internal/auth/middleware.go`)
- `RequireAuth()` gin middleware
- Extract `Authorization: Bearer <token>` header
- Parse token → set `userID` in Gin context via `c.Set("userID", id)`
- Return 401 if invalid/missing

### 2.3 — Auth Handlers (`internal/auth/handler.go`)

| Endpoint | Handler | Logic |
|----------|---------|-------|
| `POST /api/auth/register` | `Register` | Validate body → check duplicate email/username → bcrypt hash → create user → return JWT + user |
| `POST /api/auth/login` | `Login` | Find user by email → compare bcrypt → return JWT + user |
| `GET /api/auth/me` | `GetMe` 🔒 | Read `userID` from context → return user object |

### 2.4 — GORM User Model
```go
type User struct {
    ID        uuid.UUID `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    Username  string    `gorm:"uniqueIndex;size:30;not null"`
    Email     string    `gorm:"uniqueIndex;size:255;not null"`
    Password  string    `gorm:"size:255;not null"`
    AvatarURL string
    Bio       string
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

---

## Phase 3: Link CRUD & Feed

> **Goal:** Submit links, fetch feed with filters/sorting, OG metadata scraping.

### 3.1 — OG Fetcher (`internal/links/og_fetcher.go`)
- Use Colly to scrape `og:title`, `og:description`, `og:image` from submitted URLs
- Fallback to `<title>` tag if OG tags missing
- Upload OG image to Cloudinary → store `thumbnail_url`

### 3.2 — Link Repository (`internal/links/repository.go`)
- `Create(link *Link) error`
- `GetByID(id uuid.UUID) (*Link, error)` — preload User
- `GetFeed(category, tag, sort string, page, limit int) ([]Link, int64, error)`
- `Update(link *Link) error`
- `Delete(id uuid.UUID) error`
- `GetByUserID(userID uuid.UUID, page, limit int) ([]Link, int64, error)`

### 3.3 — Link Service (`internal/links/service.go`)
- `CreateLink()` — validate → fetch OG → upload thumbnail → save to DB
- `VoteLink()` — toggle logic (same vote removes it, different vote switches)
- `SaveLink()` / `UnsaveLink()`

### 3.4 — Link Handlers (`internal/links/handler.go`)

| Endpoint | Auth | Description |
|----------|------|-------------|
| `GET /api/links` | — | Paginated feed with `?category=&tag=&sort=newest|top&page=&limit=` |
| `POST /api/links` | 🔒 | Submit link (auto-fetches OG metadata) |
| `GET /api/links/:id` | — | Single link detail |
| `PUT /api/links/:id` | 🔒 owner | Edit link |
| `DELETE /api/links/:id` | 🔒 owner | Delete link |
| `POST /api/links/:id/vote` | 🔒 | Vote `{value: 1 | -1}` |
| `POST /api/links/:id/save` | 🔒 | Save link |
| `DELETE /api/links/:id/save` | 🔒 | Unsave link |

### 3.5 — GORM Models: Link, Vote, Save
- Link: UUID PK, FK to User, url, title, description, category, tags (pq.StringArray), thumbnail_url, upvotes counter
- Vote: UUID PK, unique(user_id, link_id), value IN(-1,1)
- Save: UUID PK, unique(user_id, link_id)

---

## Phase 4: User Profiles

> **Goal:** Public profiles, edit own profile, view saved links.

### 4.1 — User Repository (`internal/users/repository.go`)
- `GetByUsername(username string) (*User, error)`
- `UpdateProfile(userID uuid.UUID, bio, avatarURL string) error`
- `GetSavedLinks(userID uuid.UUID, page, limit int) ([]Link, int64, error)`

### 4.2 — User Service (`internal/users/service.go`)
- `UpdateProfile()` — handle avatar upload to Cloudinary if file provided

### 4.3 — User Handlers (`internal/users/handler.go`)

| Endpoint | Auth | Description |
|----------|------|-------------|
| `GET /api/users/:username` | — | Public profile |
| `PUT /api/users/me` | 🔒 | Edit bio, avatar |
| `GET /api/users/:username/links` | — | Links by user |
| `GET /api/users/me/saved` | 🔒 | Saved links list |

---

## Phase 5: Collections / Boards

> **Goal:** Create, manage, and share public collections of links.

### 5.1 — Collection Repository (`internal/collections/repository.go`)
- Full CRUD for collections
- `AddLink(collectionID, linkID uuid.UUID) error`
- `RemoveLink(collectionID, linkID uuid.UUID) error`
- `GetWithLinks(id uuid.UUID) (*Collection, error)` — preload links

### 5.2 — Collection Service (`internal/collections/service.go`)
- Ownership validation before mutate operations

### 5.3 — Collection Handlers (`internal/collections/handler.go`)

| Endpoint | Auth | Description |
|----------|------|-------------|
| `GET /api/collections` | 🔒 | My collections |
| `POST /api/collections` | 🔒 | Create board |
| `GET /api/collections/:id` | — | Board + links (if public) |
| `PUT /api/collections/:id` | 🔒 owner | Rename/edit board |
| `DELETE /api/collections/:id` | 🔒 owner | Delete board |
| `POST /api/collections/:id/links` | 🔒 | Add link to board |
| `DELETE /api/collections/:id/links/:link_id` | 🔒 | Remove link |

### 5.4 — GORM Model: Collection
- UUID PK, FK to User, name, description, is_public (default true)
- Many2Many relationship with Links via `collection_links` join table

---

## Phase 6: Search

> **Goal:** Basic full-text search using PostgreSQL ILIKE.

### 6.1 — Search Handler (`internal/search/handler.go`)

| Endpoint | Auth | Query | Description |
|----------|------|-------|-------------|
| `GET /api/search` | — | `?q=react&page=&limit=` | ILIKE on title, description, tags |

- Query: `WHERE title ILIKE '%q%' OR description ILIKE '%q%'`
- Paginated response with standard envelope

---

## Phase 7: Frontend — Auth & Routing

> **Goal:** Auth context, login/register pages, protected routes, app shell.

### 7.1 — Types (`src/types/index.ts`)
- Interfaces: `User`, `Link`, `Collection`, `ApiResponse<T>`

### 7.2 — API Client (`src/api/client.ts`)
- Axios instance with `baseURL` from `VITE_API_URL`
- Request interceptor: attach `Authorization: Bearer <token>` from `localStorage("ls_token")`

### 7.3 — API Modules
- `src/api/auth.ts` — `register()`, `login()`, `getMe()`
- `src/api/links.ts` — `getLinks()`, `createLink()`, `voteLink()`, `saveLink()`, etc.
- `src/api/collections.ts` — full CRUD
- `src/api/users.ts` — `getProfile()`, `updateProfile()`, `getSaved()`

### 7.4 — Auth Context (`src/context/AuthContext.tsx`)
- Stores `user` object + `token` in state
- Persists token in `localStorage` key `ls_token`
- Exposes: `login()`, `logout()`, `register()`, `isAuthenticated`
- On mount: check `ls_token` → call `getMe()` to restore session

### 7.5 — App Router (`src/App.tsx`)

| Route | Page | Access |
|-------|------|--------|
| `/` | `FeedPage` | Public |
| `/login` | `LoginPage` | Redirect if logged in |
| `/register` | `RegisterPage` | Redirect if logged in |
| `/submit` | `SubmitLinkPage` | 🔒 Protected |
| `/links/:id` | `LinkDetailPage` | Public |
| `/search` | `SearchPage` | Public |
| `/u/:username` | `ProfilePage` | Public |
| `/me/saved` | `SavedPage` | 🔒 Protected |
| `/collections` | `CollectionsPage` | 🔒 Protected |
| `/collections/:id` | `CollectionDetailPage` | Public if is_public |
| `*` | `NotFoundPage` | — |

### 7.6 — Pages: Login & Register
- Controlled form inputs
- Call `AuthContext.login()` / `AuthContext.register()` on submit
- Redirect to `/` on success
- Show error toast on failure

---

## Phase 8: Frontend — Feed & Link Components

> **Goal:** The core browsing experience — feed, cards, voting, saving.

### 8.1 — Components

| Component | Purpose |
|-----------|---------|
| `Navbar.tsx` | Top nav: logo, search bar, auth buttons / user avatar |
| `LinkCard.tsx` | Thumbnail, title, description, tags, vote count, save button, author |
| `LinkFeed.tsx` | Maps over links array, renders `LinkCard` list, shows `SkeletonCard` while loading |
| `CategoryBar.tsx` | Horizontal scrollable pill bar for category filtering |
| `TagPill.tsx` | Clickable tag pill, onClick sets filter |
| `VoteButton.tsx` | Up/down arrows with optimistic update via React Query mutation |
| `SaveButton.tsx` | Bookmark toggle with optimistic update |
| `UserAvatar.tsx` | Avatar image with fallback initials |
| `SearchBar.tsx` | Controlled input → navigates to `/search?q=` |
| `SkeletonCard.tsx` | Loading placeholder matching LinkCard dimensions |

### 8.2 — React Query Hooks (`src/hooks/`)
- `useLinks.ts` — `useQuery` for feed (with category/sort/page params), `useMutation` for create/vote/save
- `useCollections.ts` — `useQuery` for list, `useMutation` for CRUD
- `useUsers.ts` — `useQuery` for profile, saved links
- `useAuth.ts` — `useContext(AuthContext)` wrapper

### 8.3 — Pages
- `FeedPage` — CategoryBar + sort toggle + LinkFeed + pagination
- `SubmitLinkPage` — Form with URL input (fetch OG preview on blur), title, description, category dropdown, tags input
- `LinkDetailPage` — Full link view + vote/save + metadata
- `SearchPage` — SearchBar + results feed from `?q=` param

---

## Phase 9: Frontend — Profiles & Collections

### 9.1 — Profile Pages
- `ProfilePage` (`/u/:username`) — Avatar, bio, user's submitted links
- `SavedPage` (`/me/saved`) — Protected, shows saved links feed

### 9.2 — Collection Pages
- `CollectionsPage` (`/collections`) — List user's boards + create new board form
- `CollectionDetailPage` (`/collections/:id`) — Board name/description + links in it

---

## Phase 10: Polish & Production

### 10.1 — UX Polish
- Loading skeletons on all data-fetching pages
- Toast notifications for success/error actions
- Error boundaries for graceful crash recovery
- Mobile responsive layout (Tailwind breakpoints)
- Empty states (no links, no saved items, no collections)

### 10.2 — Security Hardening
- Input sanitization on all user inputs
- Rate limiting on auth endpoints (stricter: 10 req/min)
- CORS properly locked to `FRONTEND_URL` in production
- Password strength validation (min 8 chars)

### 10.3 — Production Deployment
- **Backend → Railway**: Set all env vars, deploy Go binary
- **Frontend → Vercel**: Set `VITE_API_URL` to production backend URL
- **Database → Neon**: PostgreSQL with SSL
- **Redis → Upstash**: For rate limiting and caching
- **Images → Cloudinary**: Thumbnails and avatars

---

## Build Order Summary

```
Phase 0  →  Project Setup (Go mod, Vite, deps)
Phase 1  →  Backend Infrastructure (config, DB, Redis, middleware, server)
Phase 2  →  Auth System (JWT, register, login, middleware)
Phase 3  →  Link CRUD & Feed (OG fetch, CRUD, vote, save)
Phase 4  →  User Profiles (public profile, edit, saved)
Phase 5  →  Collections (CRUD, add/remove links)
Phase 6  →  Search (ILIKE query)
Phase 7  →  Frontend Auth & Routing (context, pages, router)
Phase 8  →  Frontend Feed & Components (cards, feed, voting)
Phase 9  →  Frontend Profiles & Collections
Phase 10 →  Polish & Deploy
```

---

## Valid Categories

```
learning | web-development | dsa | ai-ml | cybersecurity | devops | ui-ux |
ai-tools | job-career | entertainment | useful-websites | open-source | other
```

---

## API Response Envelope

All endpoints return:
```json
{
  "success": true,
  "data": { },
  "error": "",
  "pagination": { "page": 1, "limit": 20, "total": 100 }
}
```

---

## Coding Conventions

### Backend (Go)
- Handlers named by verb + resource: `CreateLink`, `GetLink`, `VoteLink`
- Errors as JSON: `c.JSON(status, gin.H{"success": false, "error": "msg"})`
- Auth user: `userID := c.MustGet("userID").(uuid.UUID)`
- Validate with `go-playground/validator` struct tags
- Pagination: default `page=1, limit=20`, max `limit=100`

### Frontend (React + TS)
- All API calls via `src/api/` — never call axios from components
- All server state via React Query — no `useState` for fetched data
- Optimistic updates for vote/save
- Controlled inputs for all forms
- `LinkCard` receives `Link` as prop — no data fetching inside

---

*End of Plan — LinkSphere V1*
