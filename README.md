# StudyMate — Backend

> The API for StudyMate, an AI-powered study workspace. Students organize their course materials in one place and turn them into searchable knowledge, AI explanations with sources, flashcards, quizzes, and measurable progress.

**Current phase:** 1 of 7: Foundation (database, authentication, courses)

StudyMate is split into two repositories:

| Repo | What it contains |
| --- | --- |
| **studymate-backend** (this repo) | NestJS API, database, migrations, Docker Compose, later the worker |
| [studymate-frontend](https://github.com/RiePham/studymate-frontend) | Next.js app |

**This repo is the source of truth for the API contract.** When an endpoint changes here, update the [API reference](#api-reference) and tell the frontend side.

---

## Table of contents

1. [Why StudyMate](#why-studymate)
2. [Architecture](#architecture)
3. [Tech stack](#tech-stack)
4. [Repository structure](#repository-structure)
5. [Getting started](#getting-started)
6. [Environment variables](#environment-variables)
7. [Common commands](#common-commands)
8. [API reference](#api-reference)
9. [Data model](#data-model)
10. [Conventions](#conventions)
11. [Git workflow](#git-workflow)
12. [Testing and verification](#testing-and-verification)
13. [Troubleshooting](#troubleshooting)
14. [Roadmap](#roadmap)

---

## Why StudyMate

**The problem:** study materials are scattered across the LMS, cloud storage, and chat apps. Students spend more time finding material and hand-making flashcards and practice tests than actually studying.

**The product:** a unified workspace where a student creates a course, uploads its materials, and the system turns them into something they can search, ask questions about, and practice with.

**Guiding principle:**

> Use AI where language understanding helps. Use deterministic software where correctness matters.

StudyMate is a normal full-stack application **with an AI subsystem**, not a chatbot with a UI attached.

**Core user flow:** register / log in → create a course → upload documents → system processes them → ask the AI Tutor, generate flashcards / quizzes → review → track progress.

**MVP:** authentication · courses · upload + storage · PDF text extraction · basic document search · RAG AI Tutor · flashcards · simple quizzes · basic dashboard

**Later:** spaced repetition (SM-2) · advanced analytics · study sessions · study groups · calendar and notifications · adaptive quizzes · mobile · OCR

---

## Architecture

```
studymate-frontend (Next.js, localhost:3000)
  │   HTTP requests with credentials: 'include' (auth travels as an httpOnly cookie)
  ▼
┌─────────────────────────── this repo ───────────────────────────┐
│ NestJS API  (localhost:4000/api)                                │
│   auth, authorization, validation                               │
│   the ONLY layer allowed to read/write data                     │
│   │                                                             │
│   ├── fast path ──▶ PostgreSQL + pgvector  (localhost:5432)     │
│   │                   relational data + chunk embeddings        │
│   │                                                             │
│   └── slow path ──▶ Redis (job queue) ──▶ BullMQ worker  [3+]   │
└──────────────────────────────────────────────┬──────────────────┘
                                               ├─▶ Object storage (S3/R2)   original PDFs
                                               ├─▶ Embedding API            text → vectors
                                               └─▶ LLM API                  answers, flashcards, quizzes
```

- **Fast path:** request → API → database → response (e.g. list my courses).
- **Slow path:** the API pushes a job to Redis and returns immediately; the worker does the heavy work (text extraction, chunking, embeddings) and writes results back to the database.

| Component | Responsibility |
| --- | --- |
| Backend API (NestJS) | The single gateway to data: authentication, authorization, validation, coordination. All security checks live here. |
| PostgreSQL + pgvector | Relational data and vector embeddings of document chunks, in the same database. |
| Redis | Job queue for background work; later also caching and rate limiting. |
| Worker (BullMQ) | Separate process for heavy jobs. Scales independently; jobs retry with backoff and must be idempotent. |
| Object storage (S3/R2) | Stores original files. The database only keeps a `storage_key`. |
| AI services | Embedding model (text → vector) and LLM, called through a provider API. |

---

## Tech stack

| Layer | Technology | Phase |
| --- | --- | --- |
| Language | TypeScript | 1 |
| Framework | NestJS + TypeORM | 1 |
| Database | PostgreSQL + pgvector, via Docker Compose | 1 |
| Auth | bcrypt (cost 12) + JWT in an httpOnly cookie | 1 |
| Queue | Redis + BullMQ | 3 |
| File storage | S3 / Cloudflare R2 | 3 |
| AI | Embedding model + LLM via provider API | 4 |
| Deployment | Railway / Render / AWS, managed PostgreSQL and Redis | 7 |

---

## Repository structure

```
studymate-backend/
├── src/
│   ├── auth/             # register, login, logout, JwtAuthGuard, @CurrentUser
│   ├── users/            # User entity and service
│   ├── courses/          # Course entity, controller, service, DTOs
│   ├── common/           # shared decorators, guards, filters
│   ├── migrations/       # TypeORM migrations (schema history)
│   ├── data-source.ts    # TypeORM config for the CLI (runs outside Nest)
│   └── main.ts           # global prefix /api, ValidationPipe, cookie-parser, CORS
├── docker-compose.yml    # PostgreSQL + pgvector (volume + healthcheck)
├── .env                  # NOT committed
├── .env.example          # committed: template for others
├── .gitignore
├── CLAUDE.md             # instructions for Claude Code
└── README.md
```

---

## Getting started

### 1. Prerequisites

Install **Node.js (LTS)**, **Docker Desktop**, **Git**, and an editor. Then verify:

```bash
node --version
npm --version
git --version
docker --version
docker run --rm hello-world     # confirms Docker can actually run containers
```

### 2. Clone and configure

```bash
git clone https://github.com/RiePham/studymate-backend.git
cd studymate-backend
cp .env.example .env            # then open .env and fill in JWT_SECRET
```

Generate a strong `JWT_SECRET` with:

```bash
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

### 3. Start the database

```bash
docker compose up -d
docker compose ps               # wait until the database shows "healthy"
```

### 4. Install, migrate, run

```bash
npm install
npm run migration:run           # creates the tables
npm run start:dev
```

Check the API is alive: open <http://localhost:4000/api/health>.

To use the app in a browser, also set up [studymate-frontend](https://github.com/RiePham/studymate-frontend).

---

## Environment variables

Configuration changes between environments; code doesn't. Secrets are **never** hard-coded or committed. `ConfigModule` validates config on startup, and the app **refuses to start if a required variable is missing**.

| Variable | Example | Notes |
| --- | --- | --- |
| `DATABASE_URL` | `postgresql://postgres:postgres@localhost:5432/studymate` | Must match the credentials in `docker-compose.yml`. |
| `JWT_SECRET` | *(long random string)* | Required. Signs the auth token. |
| `JWT_EXPIRES_IN` | `1d` | Token lifetime. |
| `PORT` | `4000` | API port. |
| `FRONTEND_URL` | `http://localhost:3000` | Allowed CORS origin. Must be an exact origin, never `*`, because cookies are sent. |

When you add a new variable: add it to `.env.example`, to the `ConfigModule` validation, and to this table.

---

## Common commands

| Command | What it does |
| --- | --- |
| `docker compose up -d` | Start the database in the background |
| `docker compose ps` | Show container status / health |
| `docker compose logs -f` | Follow database logs |
| `docker compose down` | Stop containers (data is kept in the volume) |
| `npm run start:dev` | Run the API with hot reload |
| `npm run migration:generate -- src/migrations/<Name>` | Generate a migration from entity changes |
| `npm run migration:run` | Apply pending migrations |
| `npm run migration:revert` | Undo the most recent migration |

---

## API reference

Base URL: `http://localhost:4000/api`. Every endpoint except the public ones requires the `access_token` cookie set by login.

| Method | Endpoint | Access | Behavior |
| --- | --- | --- | --- |
| `GET` | `/health` | public | Liveness check: "is the API running?" |
| `POST` | `/auth/register` | public | Create an account. `400` invalid body, `409` email already registered. |
| `POST` | `/auth/login` | public | Sets the `access_token` httpOnly cookie. Wrong email **or** password → `401` with the same message. |
| `POST` | `/auth/logout` | auth | Clears the cookie. |
| `GET` | `/auth/me` | auth | Returns the current user. |
| `GET` | `/courses` | auth | Lists **only the current user's** courses. |
| `POST` | `/courses` | auth | Creates a course. `userId` comes from the token, never the body. |
| `PATCH` | `/courses/:id` | auth | Updates a course. Owner only; anyone else gets `404`. |
| `DELETE` | `/courses/:id` | auth | Deletes a course, returns `204`. Owner only; anyone else gets `404`. |

### Status codes

| Code | Meaning in StudyMate |
| --- | --- |
| `400` | Invalid data (failed DTO validation) |
| `401` | Not logged in, or wrong credentials |
| `404` | Doesn't exist **or isn't yours** (we never reveal which) |
| `409` | Conflict, e.g. email already registered |

### Request lifecycle

Each layer can stop the request:

```
cookie-parser → JwtAuthGuard (no valid token → 401) → ValidationPipe (bad body → 400)
→ Controller → Service → Repository → PostgreSQL
```

---

## Data model

**Current tables (Phase 1):** `users`, `courses`. `courses.user_id` is a foreign key to `users.id` (one user → many courses) and defines ownership.

**Planned (13 entities):** User, Course, Document, DocumentChunk, Conversation, Message, Flashcard, FlashcardReview, Quiz, Question, QuizAttempt, QuizAnswer, StudySession.

Design rules:

- **Ownership chain:** `user → course → everything else`.
- **`ON DELETE CASCADE`:** deleting a document deletes its chunks, so no orphaned data is left behind.
- **`document_chunks.course_id`** is an intentional denormalization, because search filters by course constantly.
- **Every quiz attempt is an event and every answer is a row.** This is the raw data that analytics is computed from.
- Original files are **never** stored in the database, only their `storage_key`.

---

## Conventions

These rules apply to everyone working on the codebase, human or AI assistant.

### Security and data access

- **Never look up a resource by `id` alone.** Ownership goes inside the query:
  ```ts
  async findOneForUser(id: string, userId: string) {
    const course = await this.repo.findOne({ where: { id, userId } });
    if (!course) throw new NotFoundException(); // 404, not 403
    return course;
  }
  ```
- **`userId` always comes from the JWT** (via `@CurrentUser`), never from the request body or URL.
- **DTOs + `ValidationPipe({ whitelist: true })`** strip unknown fields, so a client can't sneak in a `userId`.
- **Return `404`, not `403`,** for resources that don't exist or belong to someone else.
- **Every protected endpoint uses `JwtAuthGuard`.** The frontend's middleware only improves UX; real security lives here.
- **Passwords** are hashed with bcrypt (cost 12) and never logged or returned in responses.
- **CORS:** `credentials: true` with the exact frontend origin from `FRONTEND_URL`, never `*`.
- **Never commit `.env`.** Never hard-code secrets.

### Code structure

- **Thin controllers, thick services.** Controllers translate HTTP to service calls; business logic and DB access live in services.
- **One module per business domain** (auth, users, courses, later documents, chat, flashcards, quizzes, analytics, storage).
- **Dependency injection:** Nest creates and injects dependencies through constructors; don't `new` services yourself.
- **REST:** URLs are nouns; the action is the HTTP method.
- **Schema changes only through migrations:** generate → **read the generated SQL** → run → test that revert works. `synchronize: true` is never used.

### AI and background processing (Phase 3+)

- **AI** for explanations, draft flashcards/quizzes, summaries, and semantic retrieval. **Deterministic code** for auth, permissions, ownership, grading, progress, review scheduling, file status, and analytics.
- **Never process a file inside the upload request.** Save it, create the document record (`UPLOADED`), enqueue a job, and return immediately. The worker moves it to `PROCESSING` → `READY` or `FAILED`.
- **Jobs are idempotent** and retry with backoff. `FAILED` documents store a reason so the UI can offer Retry.
- **Never send whole documents to the LLM.** Retrieve the relevant chunks (filtered to the user's course) and always return sources (document id + page).
- **File downloads:** check ownership, then return a short-lived signed URL; the browser downloads straight from storage.
- **Measure before optimizing.**

---

## Git workflow

- **`main` always works.** Nobody pushes directly to it.
- **One branch per Jira ticket**, named with the ticket key: `SM-12-course-update`.
- **Small commits** with clear messages that include the ticket key: `SM-12 add UpdateCourseDto`.
- **One pull request per ticket**, describing what changed and how to test it. The mentor reviews before merge.
- **Tickets that touch both repos** get one branch and one PR in each repo, using the same ticket key so they're easy to match up. Merge the backend PR first if the frontend depends on a new endpoint.
- Run `git status` before every `git add .` to make sure `.env` isn't staged.

---

## Testing and verification

Before calling a feature "done", test it end to end.

### IDOR check with curl

Register users `a@test.com` and `b@test.com` first, then:

```bash
API=http://localhost:4000/api

# User A logs in and creates a course (note the returned id)
curl -s -c a.txt -H "Content-Type: application/json" \
  -d '{"email":"a@test.com","password":"Password123!"}' $API/auth/login
curl -s -b a.txt -H "Content-Type: application/json" \
  -d '{"title":"Linear Algebra"}' $API/courses

# User B logs in and tries to touch A's course
curl -s -c b.txt -H "Content-Type: application/json" \
  -d '{"email":"b@test.com","password":"Password123!"}' $API/auth/login
curl -i -b b.txt -X PATCH -H "Content-Type: application/json" \
  -d '{"title":"Hacked"}' $API/courses/<course-id>     # expect 404
curl -i -b b.txt -X DELETE $API/courses/<course-id>    # expect 404
```

### Checklist

- [ ] `GET /api/health` responds
- [ ] Register with a duplicate email → `409`
- [ ] Register with an invalid body → `400`; extra fields like `userId` are stripped
- [ ] Login with a wrong password → `401` with the same message as an unknown email
- [ ] Protected endpoints without the cookie → `401`
- [ ] User B cannot update or delete user A's course → `404`

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| App crashes immediately on start | Missing environment variable | Compare `.env` with `.env.example` |
| `ECONNREFUSED ... 5432` | Database container isn't running | `docker compose up -d`, then `docker compose ps` |
| `relation "users" does not exist` | Migrations haven't run | `npm run migration:run` |
| `type "vector" does not exist` | pgvector extension not enabled | Enable it in psql: `CREATE EXTENSION IF NOT EXISTS vector;` |
| Frontend gets CORS errors or the cookie isn't sent | CORS misconfigured | `credentials: true` and `origin` set to the exact `FRONTEND_URL` |

---

## Roadmap

| Phase | Name | Backend scope |
| --- | --- | --- |
| **1** | **Foundation** ← current | Project setup, database, migrations, authentication |
| 2 | Course system | Course CRUD, authorization |
| 3 | Documents | Upload, object storage, processing states, Redis + BullMQ worker |
| 4 | RAG | Text extraction, chunking, embeddings, vector search, AI Tutor endpoint |
| 5 | Study features | Flashcards, quizzes, attempts |
| 6 | Analytics | Progress and weak-topic queries |
| 7 | Production quality | Testing, caching, rate limiting, error handling, deployment, monitoring |

### Phase 1 checklist

- [ ] NestJS project with `.gitignore` from the first commit
- [ ] PostgreSQL + pgvector via Docker Compose, with volume and healthcheck
- [ ] Config via environment variables; app stops if a variable is missing
- [ ] `GET /api/health`
- [ ] `users` and `courses` entities with an `Init` migration (no `synchronize`)
- [ ] Register / login / logout with bcrypt and JWT in an httpOnly cookie
- [ ] `JwtAuthGuard`, `@CurrentUser`, `ValidationPipe` whitelist
- [ ] Course CRUD with ownership in every query, verified with the IDOR check
- [ ] A README that lets anyone clone and run the project

### Known security gaps (planned)

- Rate limiting on `/auth/login`, in Phase 7
- `Secure` cookie flag + HTTPS, at deployment
