# Architecture

Status: agreed design, pre-implementation.
Last updated: 2026-09-25.

This document describes the architecture of the Job Search Copilot. It is the
authoritative design reference for the initial version. If the implementation and
this document disagree, the disagreement is a bug in one of them — fix it in the
same change.

Related documents (to be written as their areas land):

- `docs/data-model.md` — field-level schema and migration notes
- `docs/api.md` — endpoint reference
- `docs/ai.md` — prompts, providers, configuration, data-handling notes

---

## 1. Goals and constraints

The application is a local-first, single-user job search command center. It
captures job postings, tracks applications through the hiring pipeline, manages
recruiters and follow-ups, records interview notes, generates interview prep,
and reports conversion analytics.

Constraints that shape every decision below:

- **Local-first.** One user, one machine, one database. Postgres via Docker
  Compose for development, with a containerised production-shaped run.
- **Single-user.** No authentication, no authorization, no user table, no
  multi-tenancy. These are not deferred features; they are out of scope. Adding
  them later is a deliberate, breaking change.
- **Small surface.** The design favours a boring, obvious implementation over
  extensibility. No plugin architecture, no event bus, no multi-module Maven
  build, no shared-component library.
- **No premature scale.** All data volumes are single-digit thousands of rows.
  Analytics is computed in-process; there is no reporting database, no cache
  tier, and no read replica.
- **Secrets stay server-side.** The browser never holds an AI provider key.

---

## 2. Repository structure

```
job-search-tracker/
├── README.md                    # product overview and stack
├── AGENTS.md                    # contributor instructions
├── .gitignore
├── .env.example                 # documented local environment variables
├── .github/workflows/ci.yml     # backend + frontend jobs
│
├── backend/
│   ├── mvnw, mvnw.cmd, .mvn/    # Maven wrapper — always use ./mvnw
│   ├── pom.xml
│   ├── Dockerfile
│   └── src/
│       ├── main/java/com/jobcopilot/...
│       ├── main/resources/
│       │   ├── application.yml
│       │   ├── application-local.yml
│       │   ├── db/migration/    # SQL migrations
│       │   └── prompts/         # AI prompt templates, reviewable in git
│       └── test/java/com/jobcopilot/...
│
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── Dockerfile
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── api/  components/  pages/  hooks/  lib/  styles/
│
├── infra/
│   └── docker-compose.yml       # postgres (+ backend, + frontend)
│
└── docs/
    ├── architecture.md          # this document
    ├── data-model.md
    ├── api.md
    └── ai.md
```

Rules:

- Exactly one Compose file, in `infra/`. Not per-service Compose files.
- The Maven wrapper is committed. `mvn` is never invoked directly.
- AI prompt templates live on disk under `src/main/resources/prompts/` rather
  than as string constants, so prompt changes show up in review diffs.
- `docs/` is updated in the same change as the code it describes.

---

## 3. Backend package structure

Base package `com.jobcopilot`, organised **by feature slice** rather than by
technical layer. At this size, a layer-based `controller/ service/ repository/`
split scatters every feature across four packages and makes the code harder to
read, not easier.

```
com.jobcopilot
├── JobCopilotApplication.java
├── shared/
│   ├── config/          # CORS, Jackson, AI properties
│   ├── error/           # GlobalExceptionHandler, ProblemDetail mapping
│   └── web/             # PageResponse<T> and shared DTO helpers
├── job/                 # Job, JobStageHistory, stage rules
├── contact/             # Contact (recruiter / hiring manager)
├── interview/           # Interview, InterviewNote
├── followup/            # FollowUp
├── analytics/           # read-only projections and aggregation queries
├── export/              # CSV writers
└── ai/                  # ports and adapters — see section 8
    ├── JobDescriptionParser.java
    ├── InterviewPrepGenerator.java
    ├── heuristic/
    └── llm/
```

Each feature slice follows the same shape:

```
job/
├── Job.java, JobStageHistory.java          # entities
├── JobRepository.java                       # Spring Data JPA
├── JobService.java                          # business logic, @Transactional
├── JobController.java                       # HTTP boundary
└── dto/                                     # request/response records
```

Conventions:

- Entities never leave the service layer. Controllers return DTO `record`s only.
- Entities are allowed to be "dumb" — fields plus accessors. Business rules live
  in services, not in entity callbacks.
- `@Transactional` is applied at the service method boundary, not the repository.
- `shared/` holds only genuinely cross-cutting code. The moment a helper is used
  by one feature, it moves into that feature's package.

---

## 4. Domain model

### 4.1 The central modelling decision

**`Job` and `Application` are a single entity in v1.** One row means "this role,
at this company, and here is where I am with it" — the description, the stage,
and the lifecycle all live together.

This is the biggest simplification in the design and the one most likely to be
regretted, so the migration path is written down in advance:

> **If separate `Job` and `Application` entities become necessary** (re-applying
> to the same role, a company freeze thawing, tracking a referral and a direct
> application separately): add an `application` table, move `stage`,
> `stageChangedAt`, `appliedAt`, and all `job_stage_history` rows onto it, leave
> `Job` as the immutable description snapshot, and introduce a `job_id` on
> `Contact`, `Interview`, and `FollowUp`. Read endpoints gain an optional
> `applicationId` filter. No field is lost, so the migration is mechanical.

### 4.2 Entities

```
Job
  id, companyName, roleTitle, descriptionRaw, descriptionUrl, source
  location, workMode, employmentType, seniority
  salaryMin, salaryMax, salaryCurrency, salaryPeriod
  requiredSkills[], niceToHaveSkills[]
  stage, stageChangedAt, appliedAt, archived
  createdAt, updatedAt

JobStageHistory        (Job 1 ── *)
  id, jobId, fromStage, toStage, note, changedAt

Contact                (Job 1 ── *)
  id, jobId, name, title, type
  email, phone, linkedinUrl, notes

Interview              (Job 1 ── *)
  id, jobId, type, scheduledAt, durationMinutes, mode, status
  interviewerNames, meetingUrl
  prepContent, prepGeneratedAt, prepModel
  createdAt, updatedAt

InterviewNote          (Interview 1 ── *)
  id, interviewId, question, answer, topics[], rating, createdAt

FollowUp               (Job 1 ── *)
  id, jobId, dueDate, channel, note, status, completedAt, createdAt
```

### 4.3 Enumerations

| Enum | Values |
|---|---|
| `JobStage` | `SAVED`, `APPLIED`, `SCREENING`, `INTERVIEW`, `FINAL_ROUND`, `OFFER`, `REJECTED`, `WITHDRAWN` |
| `WorkMode` | `REMOTE`, `HYBRID`, `ONSITE`, `UNKNOWN` |
| `EmploymentType` | `FULL_TIME`, `PART_TIME`, `CONTRACT`, `INTERNSHIP`, `UNKNOWN` |
| `SalaryPeriod` | `HOURLY`, `MONTHLY`, `YEARLY`, `UNKNOWN` |
| `ContactType` | `RECRUITER`, `HIRING_MANAGER`, `REFERRER`, `OTHER` |
| `InterviewType` | `PHONE_SCREEN`, `TECHNICAL`, `BEHAVIORAL`, `SYSTEM_DESIGN`, `ONSITE`, `FINAL`, `OFFER`, `OTHER` |
| `InterviewMode` | `VIDEO`, `PHONE`, `ONSITE`, `OTHER` |
| `InterviewStatus` | `SCHEDULED`, `COMPLETED`, `CANCELLED`, `NO_SHOW` |
| `FollowUpChannel` | `EMAIL`, `CALL`, `LINKEDIN`, `OTHER` |
| `FollowUpStatus` | `PENDING`, `DONE`, `SKIPPED` |

`Job.source` is a free-text `String` (e.g. `LinkedIn`, `Referral`, `Company
site`) rather than an enum. Source names vary and are not worth a migration
cycle.

Stage transitions are **not** hard-enforced in v1. Real pipelines skip and repeat
stages, and blocking a move would be a worse failure than a slightly odd history.
The service records history on every move and exposes an ordered stage list the
UI uses to suggest the next stage.

### 4.4 Scoping decisions

- `requiredSkills` and `niceToHaveSkills` are stored as PostgreSQL `text[]`
  columns using Hibernate `@JdbcTypeCode(SqlTypes.ARRAY)`. A `job_skill` join
  table only pays for itself once skill-based analytics ("which skill appears
  across senior roles") is wanted, which is a stretch goal, not v1.
- `Contact` is owned by a `Job`. A recruiter interviewing for four roles at the
  same company becomes four contact rows. That duplication is an accepted cost;
  the upgrade path is a `job_contact` join table.
- `InterviewPrep` is generated per interview, not per job, because prep for a
  phone screen differs from prep for an onsite loop. The generated text is
  persisted alongside the interview so it remains readable after a model or
  prompt change.
- `archived` is a flag, not a soft delete. Rejected and withdrawn rows stay
  queryable and continue to feed analytics.

### 4.5 Explicitly out of scope

Resumes, cover letters, offer and compensation negotiation, file attachments and
uploads, networking graphs, saved searches, contacts shared across jobs,
duplicate-JD detection, multi-user support, authentication.

---

## 5. Database

### 5.1 Relationships

```
        ┌────────────────────┐
        │        jobs        │   aggregate root
        └─────────┬──────────┘
                  │  FK job_id, ON DELETE CASCADE
    ┌──────────┬──┴────────┬──────────────┐
    ▼          ▼           ▼              ▼
job_stage_  contacts   interviews    follow_ups
history                 │
                        │  FK interview_id, ON DELETE CASCADE
                        ▼
                interview_notes
```

### 5.2 Tables

Plural, `snake_case`. The aggregate root table is `jobs` (not `job`).

| Table | Purpose |
|---|---|
| `jobs` | Description snapshot plus current stage and lifecycle dates |
| `job_stage_history` | Append-only record of every stage change |
| `contacts` | Recruiters, hiring managers, referrers per job |
| `interviews` | Scheduled and completed interviews, plus generated prep |
| `interview_notes` | Per-interview question and answer notes |
| `follow_ups` | Dated follow-up actions with a completion state |

### 5.3 Conventions

- Primary keys use database-generated identity.
- Timestamps are `Instant` (`timestamptz`) and always stored and served in UTC.
- Enums are stored as `varchar(32)` via `@Enumerated(EnumType.STRING)`. Adding
  a value is then a code change rather than a migration, and no PostgreSQL enum
  type has to be altered. Avoid native PG enums.
- Money is `numeric(12,2)` with `salary_currency char(3)`. Amounts are stored
  exactly as extracted, in the currency and period stated in the posting. No
  cross-currency normalisation.
- Long free text (`description_raw`, `prep_content`) is `text`.
- All child foreign keys are `ON DELETE CASCADE`. Deleting a job is a hard
  delete of its children in one transaction.
- Long-lived, structured columns are `NOT NULL` only when the domain genuinely
  requires them. Over-constraining nullable-by-nature fields (`meeting_url`,
  `applied_at`) causes constant migration churn.

### 5.4 Indexes

```
jobs(archived, stage)
jobs(company_name)
jobs(applied_at)
jobs(created_at)
job_stage_history(job_id, changed_at)
interviews(job_id, scheduled_at)
follow_ups(due_date)          WHERE status = 'PENDING'
```

The partial index on pending follow-ups is deliberate: the overdue-and-upcoming
query is the hottest read path in the application, and it only ever looks at
pending rows.

### 5.5 Schema management

The schema is owned by versioned SQL migrations, applied automatically at
startup, with Hibernate set to validate rather than generate. See
[Open decision D1](#open-decisions) — the chosen migration tool still needs to be
confirmed, and `README.md` must be updated in the same change if it adds a
dependency.

---

## 6. REST API boundaries

Base path `/api`. No version prefix: there is one client and one version, and a
version segment would be ceremony until a second client exists.

### 6.1 Conventions

- Request and response bodies are `record` DTOs, validated with
  `jakarta.validation`.
- Errors are RFC 7807 `ProblemDetail`, with an additional machine-readable
  `code` field so the frontend can branch without string-matching messages.
- `201 Created` with a `Location` header on creation; `200` with the updated
  DTO on update; `204 No Content` on delete.
- Timestamps are ISO-8601 UTC. Money is sent as a decimal string plus a currency
  code.
- Updates are partial (`PATCH`) with explicit nullable fields; there is no
  implicit merge of absent fields.
- Entity types never appear in a response body.

### 6.2 Jobs

```
GET    /api/jobs?query=&stage=&companyName=&archived=&page=&size=&sort=appliedAt,desc
POST   /api/jobs
GET    /api/jobs/{id}
PATCH  /api/jobs/{id}
DELETE /api/jobs/{id}
POST   /api/jobs/{id}/stage            body: { stage, note }
GET    /api/jobs/{id}/stage-history
```

### 6.3 Extraction

```
POST   /api/jobs/parse                 body: { descriptionText, descriptionUrl? }
```

Returns a `ParsedJobDraft` that has **not** been persisted, alongside the
provider that produced it (`llm` or `heuristic`).

Separating parse from save is a deliberate boundary. It lets the user review and
correct an extraction before anything is written, allows re-parsing a pasted
description without touching saved data, and keeps the AI adapter read-only with
respect to the database.

### 6.4 Contacts, interviews, follow-ups

Creation is scoped under the parent job, so the server never has to trust a
`jobId` supplied in the body. Updates address the child directly.

```
GET    POST    /api/jobs/{jobId}/contacts
PATCH  DELETE  /api/contacts/{id}

GET    POST    /api/jobs/{jobId}/interviews
PATCH  DELETE  /api/interviews/{id}
POST   PATCH   DELETE /api/interviews/{id}/notes
POST           /api/interviews/{id}/prep

GET    POST    /api/jobs/{jobId}/follow-ups
PATCH  DELETE  /api/follow-ups/{id}
GET           /api/follow-ups?status=PENDING&dueBefore=
```

`GET /api/follow-ups` is cross-job by design: the follow-up inbox is a
single-user to-do list, not a per-job view.

### 6.5 Analytics

```
GET /api/analytics/summary
GET /api/analytics/funnel
GET /api/analytics/activity?months=6
```

Read-only aggregations computed in-process over the full dataset. At
single-digit-thousand row volumes this is simpler and faster than maintaining
incremental counters, and a cached counter table would be wrong every time a row
is edited by hand.

`summary` returns totals by stage, applications this week and month, overdue
follow-up count, and conversion rates (response, interview, offer) with average
days to first response. `funnel` returns ordered stage counts with stage-to-stage
conversion. `activity` returns a weekly application and interview time series.

### 6.6 Export

```
GET /api/exports/jobs.csv?from=&to=&stage=
GET /api/exports/interviews.csv
GET /api/exports/follow-ups.csv
```

`text/csv` with RFC 4180 quoting and a `Content-Disposition: attachment` header,
written in-memory through a `BufferedWriter`. No streaming library, no temp
files — these exports are small by construction.

### 6.7 Operations

`/actuator/health` for container and database health. A small
`GET /api/config` returns `{ aiEnabled, extractionProvider }` so the UI can
display AI status without ever receiving the provider key.

Cross-origin requests are permitted in development for the Vite dev server
origin only. There are no cookies and no credentials, so there is no CSRF
surface.

---

## 7. Frontend structure

A React SPA that is a thin client over the API. No business rules, no derived
state that the server owns, no client-side data warehouse.

### 7.1 Libraries

Chosen for the smallest set that removes real boilerplate:

| Concern | Choice |
|---|---|
| Build | Vite |
| Routing | `react-router` |
| Server state | `@tanstack/react-query` |
| Forms and validation | `react-hook-form` + `zod` |
| Styling | Tailwind CSS v4 via `@tailwindcss/vite` |
| Charts | Recharts |
| Dates | `date-fns` |
| Tests | Vitest, React Testing Library, MSW |

There is deliberately **no global client state library**. No Redux, no Zustand.
Server data lives in the query cache; everything else is component state. If
that ever stops being true, the problem is a missing endpoint, not a missing
store.

### 7.2 Routes

| Route | Page |
|---|---|
| `/` | Dashboard — pipeline snapshot, due follow-ups, headline metrics |
| `/jobs` | Jobs list, table and kanban toggle |
| `/jobs/new` | Paste description → parse → review → save |
| `/jobs/:id` | Job detail, tabbed |
| `/jobs/:id/edit` | Full edit form |
| `/interviews` | Upcoming and past interviews across all jobs |
| `/interviews/:id` | Interview detail: details, prep, notes |
| `/follow-ups` | Follow-up inbox: overdue, today, upcoming, done |
| `/analytics` | Funnel, conversion, activity |
| `/settings` | AI status, data export, sample data |

### 7.3 Source layout

```
src/
├── main.tsx, App.tsx, routes.tsx
├── api/            client.ts (fetch wrapper, ProblemDetail → typed Error),
│                   jobs.ts, contacts.ts, interviews.ts, followUps.ts,
│                   analytics.ts, exports.ts
├── hooks/          useJobs, useJob, useDebouncedQuery, useFollowUps, useAnalytics
├── lib/            types.ts, queryKeys.ts, format.ts, stages.ts
├── components/
│   ├── layout/     AppShell, Sidebar, TopBar
│   ├── ui/         Button, Input, Select, Textarea, Modal, Badge, Card, Table,
│   │               EmptyState, Spinner, Toast, DateField, ConfirmDialog
│   ├── jobs/       JobTable, JobFilters, StageBadge, StageSelect, KanbanBoard,
│   │               JobForm, JobDescriptionInput, ParseReviewPanel, StageTimeline
│   ├── contacts/   ContactList, ContactForm
│   ├── interviews/ InterviewForm, InterviewList, PrepPanel, NoteList, NoteForm
│   ├── followups/  FollowUpList, FollowUpForm, DueBadge
│   └── analytics/  StatCard, FunnelChart, ActivityChart, StageBreakdown
├── pages/          one file per route
└── styles/         index.css (Tailwind entry and theme tokens)
```

The kanban view is a presentation alternative to the table over the same query
results. It ships after the table, and only if it earns its place.

### 7.4 Data conventions

- Every server read is a query with a key from a single `queryKeys` factory.
- Every mutation invalidates the keys it affects; no manual cache surgery.
- The query key always includes every parameter that changes the result set,
  including pagination and sort.
- Search inputs are debounced before they reach a query key.
- Errors surface through the `ProblemDetail` title and detail, never as a raw
  stack trace or a silent console log.

### 7.5 Local development

Vite dev server proxies `/api` to `http://localhost:8080`, so the browser never
makes a cross-origin request during development and CORS stays out of the
debugging path.

---

## 8. AI functionality

**All AI capability lives in the backend.** The browser never sees a provider
key, there is exactly one implementation to maintain and test, and the endpoints
are reusable from scripts or future automation.

### 8.1 Ports

Two capability interfaces in `com.jobcopilot.ai`, both owned by the application
and both free of any provider vocabulary:

```
JobDescriptionParser
  ParsedJob parse(String descriptionText)

InterviewPrepGenerator
  InterviewPrep generate(PrepRequest request)   // job + interview + optional notes
```

The rest of the backend depends only on these interfaces. A controller, a
service, or a test can substitute an implementation without knowing whether a
language model was involved.

### 8.2 Adapters

- **`heuristic/HeuristicJobDescriptionParser`** — section-header detection
  ("Requirements", "Responsibilities"), labelled-field parsing, salary and
  location pattern matching, and a skill dictionary. It needs no API key, is
  fully deterministic, is exhaustively unit-testable, and is the default
  implementation.
- **`llm/OpenAiCompatible*`** — a `RestClient` against any chat-completions
  endpoint, configured by `AI_BASE_URL`, `AI_API_KEY`, and `AI_MODEL`. Outputs
  are requested as schema-constrained JSON and deserialised into the same DTO
  records the heuristic adapter returns.

Talking to an OpenAI-compatible endpoint is an implementation detail that covers
OpenAI, most hosted providers, and local runtimes such as Ollama. Only the
default configuration differs between them.

### 8.3 Configuration and fallback

The provider key is read from the environment (via `.env` for local
development). It is never stored in the database and never returned to the
browser. `GET /api/config` reports only whether AI is enabled and which provider
is active.

If no key is configured, extraction still works through the heuristic adapter and
the response states which provider produced it, so the UI can show "basic
extraction — no AI key configured" instead of silently degrading.

### 8.4 Data handling

Job descriptions and interview notes leave the machine when the LLM path is
used. The active provider and model are user-visible in Settings, and `docs/ai.md`
records this. Running against a local model is a configuration change, not a
code change.

### 8.5 Out of scope

Embeddings or retrieval over past notes, automatic stage changes inferred from
text, resume tailoring, and cover-letter generation. Each of these expands the
blast radius of a probabilistic system inside an app whose value depends on the
user trusting their own records.

---

## 9. Testing strategy

### 9.1 Backend

| Level | Covers | Tooling |
|---|---|---|
| Unit | Stage rules, follow-up due logic, CSV escaping, heuristic parser | Plain JUnit, no Spring context |
| Web slice | Status codes, validation, `ProblemDetail` mapping, DTO mapping | `@WebMvcTest` + `MockMvc` |
| Data slice | Custom queries, array columns, cascades, filters | `@DataJpaTest` + Testcontainers |
| Integration | Flows across layers, e.g. create job → move stage → analytics reflects it | `@SpringBootTest` + Testcontainers |
| AI adapter | Request and response mapping, schema-constrained output, malformed responses | `MockRestServiceServer` or WireMock |

Decisions:

- **Testcontainers PostgreSQL, not H2.** The schema uses `text[]`, partial
  indexes, and `timestamptz` mapping. H2 in PostgreSQL-compatibility mode would
  pass tests that then fail in production. GitHub Actions runners have Docker, so
  this costs nothing extra.
- **Containers are reused across test classes** to keep the suite under a
  couple of minutes.
- **No test ever calls a real model.** The heuristic adapter is tested for real
  behaviour; the LLM adapter is tested at the HTTP boundary with a mock server.
- **A `demo` Spring profile** seeds representative data for manual exploration.
- **JaCoCo reports coverage with a 60% line gate scoped to service classes
  only.** A global gate on a project this size produces coverage theatre.

### 9.2 Frontend

Vitest and React Testing Library for component and hook behaviour, with MSW
handlers doubling as shared API fixtures. `lint`, `typecheck`, `test`, and
`build` all run in CI. Playwright is deferred to a small number of end-to-end
smoke flows late in the project, not a maintained suite.

### 9.3 CI

`.github/workflows/ci.yml` runs two parallel jobs:

- **backend** — `./mvnw -B verify` on JDK 21
- **frontend** — `npm ci && npm run lint && npm run typecheck && npm test && npm run build`

No Compose service is needed in CI because Testcontainers provides the database.

---

## 10. Deployment and operation

- `docker compose up` brings up Postgres, the backend, and the frontend from
  `infra/docker-compose.yml`. This is the supported local run and the
  verification path for releases.
- Development uses a `local` profile with a Vite dev server and HMR; production
  builds the frontend to static files served by a small reverse proxy.
- Data lives in a Postgres volume. Backing it up is `pg_dump`; the schema is
  small and entirely owned by migrations.
- There is no message broker, cache, cache-invalidation concern, or background
  scheduler. Prep generation is synchronous with an explicit loading state in
  the UI, which is honest about how long it takes and avoids building a job
  system for a single-user app.

---

## 11. Open decisions

These were raised during design and are recorded here rather than resolved by
assumption. The stated default is what implementation should use unless the
decision changes.

| # | Decision | Default in this document | Needs |
|---|---|---|---|
| D1 | Migration tool | Versioned SQL migrations, Hibernate set to `validate` | Confirmation. Adds a dependency, so `README.md` must change in the same commit per `AGENTS.md`. |
| D2 | AI provider | Any OpenAI-compatible chat-completions endpoint | The default base URL and model to ship with. |
| D3 | `Job` vs `Job` + `Application` | Single `jobs` table holding the description and the lifecycle | Confirmation, given the migration path in section 4.1. |
| D4 | Styling approach | Tailwind CSS v4 | Confirmation. CSS Modules remain a reasonable alternative. |

---

## 12. Implementation order

1. Backend bootstrap, Compose, migrations, CI
2. Job core — entity, CRUD, validation, error handling
3. Stage tracking — enum, history, stage endpoint
4. Job query API — search, filters, sorting, pagination
5. Frontend foundation — scaffold, routing, data layer, app shell, CI
6. Jobs list page
7. Job create and detail pages, including stage control and timeline
8. Contacts — API and UI
9. Follow-ups — API and UI, including the inbox
10. Interviews and notes — API and UI
11. Extraction — ports, heuristic parser, parse endpoint, review-and-save wizard
12. LLM adapter — configuration, schema-constrained output, fallback
13. Interview prep generation — API and UI
14. Analytics — aggregation API, dashboard, charts
15. CSV export, containerisation, and documentation

---

## 13. Deliberately deferred

Kanban view, saved searches, resume and cover-letter storage, application and
offer detail, duplicate-description detection, tagging and notes on jobs
themselves, multi-arch container builds, and a Playwright suite.

Each is a real idea. None is required to run a job search with this tool, and
each one that lands is cheap to add later precisely because nothing above
depends on it.
