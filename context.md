# LinkSphere — Project Context for AI Setup

> This file is the single source of truth for the LinkSphere V1 project.
> Use it to instruct an AI to scaffold, configure, or build any part of the project.
> Every section is labelled with its purpose so the AI knows exactly what to do.

---

## 1. Project Overview

**Name:** LinkSphere  
**Tagline:** A community-driven platform to discover, save, and share the best resources on the internet.  
**Type:** Full-stack web application — social link-sharing platform (like Pinterest meets Product Hunt for links).

### What it does
Users submit URLs (with title, description, category, tags). The platform auto-fetches the Open Graph preview image. Other users can upvote, save, and organise links into public collections/boards. A public feed shows all links, filterable by category and sortable by newest or most upvoted.

### V1 Scope (build only this, nothing else)
- User registration and login (JWT auth)
- Submit a link (auto-fetch OG metadata + thumbnail)
- Public feed with category filters and sorting
- Upvote / downvote links
- Save links to a personal saved list
- Public user profiles
- Collections / boards (public only)
- Basic search (PostgreSQL ILIKE)

### Explicitly OUT of V1 scope
Do NOT build: comments, notifications, follow system, AI features, recommendation engine, gamification, private/collaborative collections, broken link detection, OAuth.

---

## 2. Tech Stack

### Frontend
| Tool | Version | Purpose |
|------|---------|---------|
| React | 18 | UI framework |
| Vite | 5 | Build tool (NOT Next.js — plain SPA) |
| TypeScript | 5 | Type safety |
| Tailwind CSS | 3 | Styling |
| shadcn/ui | latest | Pre-built component library |
| React Router | v6 | Client-side routing |
| TanStack Query (React Query) | v5 | Server state + caching |
| Axios | 1 | HTTP client |

### Backend
| Tool | Version | Purpose |
|------|---------|---------|
| Go | 1.22+ | Language |
| Gin | v1 | HTTP router and framework |
| GORM | v2 | ORM for PostgreSQL |
| golang-jwt/jwt | v5 | JWT creation and validation |
| go-playground/validator | v10 | Request body validation |
| gocolly/colly | v2 | OG metadata scraping |
| go-redis/v9 | latest | Redis client |
| cloudinary-go | v2 | Image uploads |

### Infrastructure
| Service | Purpose | Tier |
|---------|---------|------|
| Neon | PostgreSQL database | Free |
| Upstash | Redis (rate limiting, cache) | Free |
| Cloudinary | Image storage (thumbnails, avatars) | Free |
| Railway | Go backend hosting | Free |
| Vercel | React frontend hosting | Free |

### Environment Variables

#### Backend `.env`
```
PORT=8080
DATABASE_URL=postgres://user:password@host/linksphere?sslmode=require
REDIS_URL=redis://...
JWT_SECRET=your-super-secret-key-min-32-chars
JWT_EXPIRY_HOURS=72
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=
FRONTEND_URL=http://localhost:5173
```

#### Frontend `.env`
```
VITE_API_URL=http://localhost:8080/api
```

---

## 3. Database Schema

Run these migrations in order. Use GORM AutoMigrate or raw SQL files in `migrations/`.

### Table: users
```sql
CREATE TABLE users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username    VARCHAR(30) UNIQUE NOT NULL,
  email       VARCHAR(255) UNIQUE NOT NULL,
  password    VARCHAR(255) NOT NULL,         -- bcrypt hash
  avatar_url  TEXT,
  bio         TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### Table: links
```sql
CREATE TABLE links (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  url           TEXT NOT NULL,
  title         VARCHAR(200) NOT NULL,
  description   TEXT,
  category      VARCHAR(50) NOT NULL,
  tags          TEXT[],                      -- PostgreSQL array
  thumbnail_url TEXT,
  upvotes       INT DEFAULT 0,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_links_category ON links(category);
CREATE INDEX idx_links_created_at ON links(created_at DESC);
CREATE INDEX idx_links_upvotes ON links(upvotes DESC);
```

### Table: votes
```sql
CREATE TABLE votes (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  link_id    UUID NOT NULL REFERENCES links(id) ON DELETE CASCADE,
  value      SMALLINT NOT NULL CHECK (value IN (-1, 1)),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, link_id)
);
```

### Table: saves
```sql
CREATE TABLE saves (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  link_id    UUID NOT NULL REFERENCES links(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, link_id)
);
```

### Table: collections
```sql
CREATE TABLE collections (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name        VARCHAR(100) NOT NULL,
  description TEXT,
  is_public   BOOLEAN DEFAULT TRUE,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### Table: collection_links
```sql
CREATE TABLE collection_links (
  collection_id UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
  link_id       UUID NOT NULL REFERENCES links(id) ON DELETE CASCADE,
  added_at      TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (collection_id, link_id)
);
```

---

## 4. API Endpoints

**Base URL:** `/api`  
**Auth header:** `Authorization: Bearer <jwt_token>`  
**🔒** = requires auth header

### Standard response envelope
```json
{
  "success": true,
  "data": { },
  "error": "",
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

### Auth routes (`/api/auth`)
| Method | Path | Auth | Body / Params | Description |
|--------|------|------|---------------|-------------|
| POST | `/auth/register` | — | `{username, email, password}` | Create account, returns JWT |
| POST | `/auth/login` | — | `{email, password}` | Returns JWT + user object |
| GET | `/auth/me` | 🔒 | — | Returns current user |

### User routes (`/api/users`)
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/users/:username` | — | Public profile |
| PUT | `/users/me` | 🔒 | Edit profile (bio, avatar_url) |
| GET | `/users/:username/links` | — | Links submitted by user |
| GET | `/users/me/saved` | 🔒 | Current user's saved links |

### Link routes (`/api/links`)
| Method | Path | Auth | Body / Query | Description |
|--------|------|------|--------------|-------------|
| GET | `/links` | — | `?category=&tag=&sort=newest\|top&page=&limit=` | Paginated feed |
| POST | `/links` | 🔒 | `{url, title, description, category, tags[]}` | Submit link |
| GET | `/links/:id` | — | — | Single link detail |
| PUT | `/links/:id` | 🔒 (owner) | `{title, description, category, tags[]}` | Edit link |
| DELETE | `/links/:id` | 🔒 (owner) | — | Delete link |
| POST | `/links/:id/vote` | 🔒 | `{value: 1 \| -1}` | Vote (toggles if same value) |
| POST | `/links/:id/save` | 🔒 | — | Save link |
| DELETE | `/links/:id/save` | 🔒 | — | Unsave link |

### Collection routes (`/api/collections`)
| Method | Path | Auth | Body | Description |
|--------|------|------|------|-------------|
| GET | `/collections` | 🔒 | — | My collections |
| POST | `/collections` | 🔒 | `{name, description}` | Create board |
| GET | `/collections/:id` | — | — | Collection + its links |
| PUT | `/collections/:id` | 🔒 (owner) | `{name, description}` | Rename board |
| DELETE | `/collections/:id` | 🔒 (owner) | — | Delete board |
| POST | `/collections/:id/links` | 🔒 | `{link_id}` | Add link to board |
| DELETE | `/collections/:id/links/:link_id` | 🔒 | — | Remove link from board |

### Search route
| Method | Path | Auth | Query | Description |
|--------|------|------|-------|-------------|
| GET | `/search` | — | `?q=react&page=&limit=` | ILIKE search on title, description, tags |

---

## 5. Folder Structure

### Frontend — `linksphere-frontend/`
```
linksphere-frontend/
├── public/
├── src/
│   ├── api/
│   │   ├── client.ts          # axios instance, base URL from env, attach JWT header
│   │   ├── auth.ts            # register(), login(), getMe()
│   │   ├── links.ts           # getLinks(), createLink(), voteLink(), saveLink(), etc.
│   │   ├── collections.ts     # all collection CRUD functions
│   │   └── users.ts           # getProfile(), updateProfile(), getSaved()
│   │
│   ├── components/
│   │   ├── ui/                # shadcn/ui auto-generated components go here
│   │   ├── Navbar.tsx         # top nav with auth state, search bar
│   │   ├── LinkCard.tsx       # the primary card: thumbnail, title, tags, vote, save
│   │   ├── LinkFeed.tsx       # maps over links array, shows skeletons while loading
│   │   ├── CategoryBar.tsx    # horizontal scrollable filter bar
│   │   ├── TagPill.tsx        # clickable tag, onClick sets filter
│   │   ├── VoteButton.tsx     # up/down with optimistic update via React Query
│   │   ├── SaveButton.tsx     # bookmark toggle
│   │   ├── UserAvatar.tsx     # avatar image with fallback initials
│   │   ├── SearchBar.tsx      # controlled input, navigates to /search?q=
│   │   └── SkeletonCard.tsx   # loading placeholder matching LinkCard dimensions
│   │
│   ├── context/
│   │   └── AuthContext.tsx    # stores user object + JWT in memory, exposes login/logout
│   │
│   ├── hooks/
│   │   ├── useAuth.ts         # useContext(AuthContext) wrapper
│   │   ├── useLinks.ts        # useQuery / useMutation for all link operations
│   │   ├── useCollections.ts  # useQuery / useMutation for collections
│   │   └── useUsers.ts        # useQuery for profile, saved links
│   │
│   ├── pages/
│   │   ├── FeedPage.tsx       # home page — feed + category bar + sort controls
│   │   ├── LoginPage.tsx      # login form, redirects to / on success
│   │   ├── RegisterPage.tsx   # register form
│   │   ├── SubmitLinkPage.tsx # submit form, shows OG preview after URL blur
│   │   ├── LinkDetailPage.tsx # single link expanded view
│   │   ├── ProfilePage.tsx    # public profile — avatar, bio, user's links
│   │   ├── SavedPage.tsx      # /me/saved — private, protected route
│   │   ├── CollectionsPage.tsx# list + create boards, view board contents
│   │   ├── SearchPage.tsx     # search results for ?q=
│   │   └── NotFoundPage.tsx   # 404 fallback
│   │
│   ├── types/
│   │   └── index.ts           # TypeScript interfaces: User, Link, Collection, Vote, ApiResponse
│   │
│   ├── utils/
│   │   ├── formatDate.ts      # "2 days ago" relative time formatter
│   │   └── truncate.ts        # truncate string to N chars with ellipsis
│   │
│   ├── App.tsx                # React Router v6 routes, QueryClientProvider, AuthProvider
│   └── main.tsx               # ReactDOM.createRoot
│
├── .env                       # VITE_API_URL=http://localhost:8080/api
├── .env.example
├── tailwind.config.ts
├── tsconfig.json
├── vite.config.ts
└── package.json
```

### Backend — `linksphere-backend/`
```
linksphere-backend/
├── cmd/
│   └── server/
│       └── main.go            # reads config, init DB+Redis, registers routes, starts server
│
├── internal/
│   ├── auth/
│   │   ├── handler.go         # POST /auth/register, POST /auth/login, GET /auth/me
│   │   ├── middleware.go      # RequireAuth gin middleware — validates JWT, sets user in context
│   │   └── jwt.go             # GenerateToken(userID), ParseToken(tokenStr) using golang-jwt
│   │
│   ├── links/
│   │   ├── handler.go         # all /links route handlers
│   │   ├── service.go         # business logic: OG fetch on create, vote toggle logic
│   │   ├── repository.go      # GORM queries for links, votes, saves
│   │   └── og_fetcher.go      # uses colly to fetch og:title, og:image, og:description
│   │
│   ├── users/
│   │   ├── handler.go         # GET /users/:username, PUT /users/me, GET /users/me/saved
│   │   ├── service.go         # avatar upload to Cloudinary
│   │   └── repository.go      # GORM queries for user profile, saved links
│   │
│   ├── collections/
│   │   ├── handler.go         # all /collections route handlers
│   │   ├── service.go         # ownership checks before mutate
│   │   └── repository.go      # GORM queries for collections + collection_links
│   │
│   ├── search/
│   │   └── handler.go         # GET /search — ILIKE on title, description; GIN-injected db
│   │
│   └── middleware/
│       ├── cors.go             # allow FRONTEND_URL origin, common headers
│       ├── ratelimit.go        # Redis sliding window — 60 req/min per IP
│       └── logger.go           # Gin request logger with response time
│
├── pkg/
│   ├── database/
│   │   └── postgres.go        # GORM open + AutoMigrate all models
│   ├── cache/
│   │   └── redis.go           # go-redis client init from REDIS_URL
│   ├── cloudinary/
│   │   └── upload.go          # UploadImage(file io.Reader) returns secure_url string
│   └── config/
│       └── config.go          # reads .env into Config struct using os.Getenv
│
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_links.sql
│   ├── 003_create_votes.sql
│   ├── 004_create_saves.sql
│   ├── 005_create_collections.sql
│   └── 006_create_collection_links.sql
│
├── .env                       # (see env vars section above)
├── .env.example
├── go.mod                     # module: github.com/yourteam/linksphere-backend
└── go.sum
```

---

## 6. Key Data Types

### TypeScript (frontend) — `src/types/index.ts`
```typescript
export interface User {
  id: string
  username: string
  email: string
  avatar_url: string | null
  bio: string | null
  created_at: string
}

export interface Link {
  id: string
  user_id: string
  user: Pick<User, 'id' | 'username' | 'avatar_url'>
  url: string
  title: string
  description: string | null
  category: string
  tags: string[]
  thumbnail_url: string | null
  upvotes: number
  user_vote: -1 | 0 | 1        // 0 = no vote from current user
  user_saved: boolean
  created_at: string
}

export interface Collection {
  id: string
  user_id: string
  name: string
  description: string | null
  is_public: boolean
  links?: Link[]
  created_at: string
}

export interface ApiResponse<T> {
  success: boolean
  data: T
  error: string
  pagination?: {
    page: number
    limit: number
    total: number
  }
}
```

### Go (backend) — GORM models (define in each feature's repository.go or a shared models package)
```go
// User model
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

// Link model
type Link struct {
    ID           uuid.UUID `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    UserID       uuid.UUID `gorm:"type:uuid;not null"`
    User         User      `gorm:"foreignKey:UserID"`
    URL          string    `gorm:"not null"`
    Title        string    `gorm:"size:200;not null"`
    Description  string
    Category     string    `gorm:"size:50;not null"`
    Tags         pq.StringArray `gorm:"type:text[]"`
    ThumbnailURL string
    Upvotes      int       `gorm:"default:0"`
    CreatedAt    time.Time
    UpdatedAt    time.Time
}

// Vote model
type Vote struct {
    ID        uuid.UUID `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    UserID    uuid.UUID `gorm:"type:uuid;not null;uniqueIndex:idx_user_link_vote"`
    LinkID    uuid.UUID `gorm:"type:uuid;not null;uniqueIndex:idx_user_link_vote"`
    Value     int16     `gorm:"not null;check:value IN (-1,1)"`
    CreatedAt time.Time
}

// Save model
type Save struct {
    ID        uuid.UUID `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    UserID    uuid.UUID `gorm:"type:uuid;not null;uniqueIndex:idx_user_link_save"`
    LinkID    uuid.UUID `gorm:"type:uuid;not null;uniqueIndex:idx_user_link_save"`
    CreatedAt time.Time
}

// Collection model
type Collection struct {
    ID          uuid.UUID `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    UserID      uuid.UUID `gorm:"type:uuid;not null"`
    Name        string    `gorm:"size:100;not null"`
    Description string
    IsPublic    bool      `gorm:"default:true"`
    Links       []Link    `gorm:"many2many:collection_links;"`
    CreatedAt   time.Time
    UpdatedAt   time.Time
}
```

---

## 7. Categories (valid values for link.category)

```
learning | web-development | dsa | ai-ml | cybersecurity | devops | ui-ux |
ai-tools | job-career | entertainment | useful-websites | open-source | other
```

---

## 8. Route Map (React Router)

```
/                    → FeedPage          (public)
/login               → LoginPage         (redirect to / if already logged in)
/register            → RegisterPage      (redirect to / if already logged in)
/submit              → SubmitLinkPage    (🔒 protected)
/links/:id           → LinkDetailPage    (public)
/search              → SearchPage        (public, reads ?q= param)
/u/:username         → ProfilePage       (public)
/me/saved            → SavedPage         (🔒 protected)
/collections         → CollectionsPage   (🔒 protected — shows own collections)
/collections/:id     → CollectionDetailPage (public if is_public=true)
*                    → NotFoundPage
```

---

## 9. Go Module Dependencies

```bash
# Run these after go mod init github.com/yourteam/linksphere-backend

go get github.com/gin-gonic/gin
go get github.com/gin-contrib/cors
go get gorm.io/gorm
go get gorm.io/driver/postgres
go get github.com/lib/pq
go get github.com/google/uuid
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto
go get github.com/go-playground/validator/v10
go get github.com/gocolly/colly/v2
go get github.com/redis/go-redis/v9
go get github.com/cloudinary/cloudinary-go/v2
go get github.com/joho/godotenv
```

---

## 10. Frontend NPM Setup

```bash
npm create vite@latest linksphere-frontend -- --template react-ts
cd linksphere-frontend

npm install \
  axios \
  react-router-dom \
  @tanstack/react-query \
  @tanstack/react-query-devtools

# Tailwind
npm install -D tailwindcss @tailwindcss/vite
# Add to vite.config.ts: plugins: [react(), tailwindcss()]
# Add to src/index.css: @import "tailwindcss";

# shadcn/ui (run after tailwind is configured)
npx shadcn@latest init
# Select: TypeScript, default style, slate base color, src/components/ui path

# Useful shadcn components to install upfront:
npx shadcn@latest add button input label card badge toast avatar skeleton
```

---

## 11. Git Branching Strategy

```
main          → production only. Requires PR + review to merge.
dev           → integration branch. All PRs merge here first.
feature/*     → one branch per feature (e.g. feature/auth, feature/link-feed)
fix/*         → bug fixes
```

**PR naming:** `[FE] Add vote button component` or `[BE] Add link vote endpoint`

---

## 12. What to Build First (Sprint Order)

An AI setting up this project should scaffold in this order:

1. **Backend first:** Init Go module → config → DB connect → user model → auth handlers → JWT middleware
2. **Frontend auth:** Init Vite → install deps → AuthContext → login/register pages → protected route wrapper
3. **Link CRUD:** Backend link model + GET/POST endpoints → Frontend feed page + LinkCard + submit form
4. **Engagement:** Vote + save endpoints → VoteButton + SaveButton with optimistic updates
5. **Profiles:** User profile endpoints → ProfilePage + SavedPage
6. **Collections:** Collections CRUD → CollectionsPage + "add to board" UI
7. **Search:** Search endpoint → SearchPage
8. **Polish:** Skeletons, toasts, error boundaries, mobile responsiveness, production deploy

---

## 13. Coding Conventions

### Backend (Go)
- Handler functions are named by HTTP verb + resource: `CreateLink`, `GetLink`, `VoteLink`
- Always return errors as JSON using a helper: `c.JSON(http.StatusBadRequest, gin.H{"success": false, "error": "message"})`
- Extract authenticated user from Gin context: `userID := c.MustGet("userID").(uuid.UUID)`
- Validate all request bodies using go-playground/validator struct tags
- Use `bcrypt.GenerateFromPassword` for passwords, never store plain text
- Pagination default: `page=1, limit=20`, max limit=100

### Frontend (React + TypeScript)
- All API calls go through functions in `src/api/`, never call axios directly from a component
- All server state uses React Query — no useState for fetched data
- Protected routes check `AuthContext.user !== null` before rendering
- Optimistic updates for vote and save (update cache immediately, rollback on error)
- All forms use controlled inputs with local useState for form fields
- `LinkCard` receives a `Link` object as prop — no data fetching inside the card

---

## 14. Setup Instructions for AI

When asked to set up this project, follow these steps:

### Backend setup task
1. Create the folder structure exactly as defined in Section 5
2. Run `go mod init github.com/yourteam/linksphere-backend`
3. Install all Go dependencies from Section 9
4. Create `pkg/config/config.go` — reads all env vars from Section 2 into a struct
5. Create `pkg/database/postgres.go` — GORM init with all models from Section 6
6. Create `pkg/cache/redis.go` — go-redis init from REDIS_URL
7. Create all 6 migration SQL files from Section 3
8. Create `internal/auth/jwt.go`, `handler.go`, `middleware.go` (register, login, me, RequireAuth)
9. Create `cmd/server/main.go` — loads config, connects DB+Redis, registers all routes, listens on PORT
10. Create `.env.example` with all keys from Section 2 (values blank)

### Frontend setup task
1. Create the folder structure exactly as defined in Section 5
2. Run the npm setup commands from Section 10
3. Create `src/types/index.ts` with all interfaces from Section 6
4. Create `src/api/client.ts` — axios instance that reads `VITE_API_URL`, attaches JWT from localStorage key `ls_token`
5. Create `src/api/auth.ts`, `links.ts`, `collections.ts`, `users.ts` with typed API functions
6. Create `src/context/AuthContext.tsx` — stores user+token, exposes login/logout/register
7. Create `src/App.tsx` with all routes from Section 8, wrapped in QueryClientProvider + AuthProvider
8. Create all page files (empty with placeholder content) from Section 5
9. Create all component files (empty with placeholder content) from Section 5
10. Create `.env` with `VITE_API_URL=http://localhost:8080/api`

---

*End of context.md — LinkSphere V1*
