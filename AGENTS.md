# AGENTS.md

## Product purpose

This is a local-first Job Search Copilot for managing applications,
job descriptions, interview preparation, recruiter follow-ups, and
job-search analytics.

The first version is single-user and local-first.
Do not add authentication or multi-user features unless explicitly requested.

## Project stack

`README.md` defines the project's technology choices. Treat these as binding unless the
user explicitly decides to change them:

- **backend/** — Java 21, Spring Boot, Spring Data JPA, PostgreSQL, JUnit
- **frontend/** — React, TypeScript, Vite
- **infra/** — Docker, Docker Compose
- **CI** — GitHub Actions

The backend build tool is Maven.

If a technology choice changes, update `README.md` in the same change so documentation
does not drift from the implementation.

## Repository layout

- `backend/` — Spring Boot application
- `frontend/` — React SPA
- `infra/` — local development and infrastructure configuration
- `docs/` — architecture and design documentation

Some directories may not exist in a fresh checkout until they contain tracked files.

## Build and verification

Use repository-provided tooling rather than assuming global tools are installed.

For the backend, generate and use the Maven wrapper:

```bash
cd backend
./mvnw test
