# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Monocle

Monocle is a development team analytics platform that indexes pull requests, merge requests, and code reviews from GitHub, GitLab, and Gerrit, plus issues from BugZilla/Jira, and provides metrics dashboards and search over that data.

Three services: **API** (serves the web UI + REST API on port 8080), **Crawler** (indexes from providers), **Elasticsearch** (data store on port 9200).

## Development Commands

All commands use `just` (reads `.env` for environment variables). Nix is the recommended dev environment.

```bash
# Services
just elastic          # Start Elasticsearch (required for tests and API)
just api              # Run API service (port 8080)
just crawler          # Run crawler service
just web              # Web dev server with hot-reload (port 13000)

# Building
just build-web        # Build frontend (cd web && npm install && npm run build)

# Testing — Elasticsearch must be running
just ci               # Full test suite (linters + tests)
just ci-fast          # Linters only (fourmolu + hlint), no Elasticsearch needed
just test "PATTERN"   # Run a single test by name pattern
just fmt              # Auto-fix formatting (fourmolu) and apply hlint hints

# Dev tools
just ghcid            # Live Haskell recompiler
just repl             # GHCi REPL (loads Monocle modules)
just docs             # Build + open Haddock docs

# Code generation (after changing .proto files or http.proto)
just codegen          # Regenerate all: Haskell types, JS types, HTTP stubs, OpenAPI
```

Without Nix: `cabal build monocle`, `cabal test`, `cabal run monocle -- api`

Frontend only (without Nix): `cd web && npm install && npm run build`

## Deploying Changes Locally (Docker)

The docker-compose setup uses a pre-built image. To test frontend changes locally without a full rebuild:

```bash
# Build only the web stage (fast, cached npm install)
docker build --platform=linux/amd64 --target web-builder -t monocle-web-builder -f DockerfileDebian .

# Copy build artifacts into the running api container
docker create --name monocle-web-tmp monocle-web-builder
docker cp monocle-web-tmp:/monocle-webapp/build /tmp/monocle-web-build
docker rm monocle-web-tmp
docker cp /tmp/monocle-web-build/. monocle-api-1:/usr/share/monocle/webapp/
```

The web stage uses `--platform=linux/amd64` because rescript/bs-platform ships only x86_64 binaries. On Apple Silicon this triggers Rosetta emulation automatically.

## Architecture

### Data Flow
```
Providers (GitHub/GitLab/Gerrit/BugZilla)
  → Crawler (Lentille/Macroscope framework)
    → API (via protobuf/HTTP)
      → Elasticsearch
        → API (query)
          → Web UI (ReScript/React)
```

### Backend (Haskell)
- `app/Monocle.hs` → `src/CLI.hs`: entry point, dispatches `api` / `crawler` subcommands
- `src/Monocle/Main.hs`: API server initialization (Servant + WAI)
- `src/Monocle/Api/`: request handlers
- `src/Monocle/Backend/`: Elasticsearch queries and document types (via Bloodhound)
- `src/Monocle/Search/`: custom search query language parser (Megaparsec)
- `src/Lentille/`: provider crawlers (GitHub GraphQL, GitLab REST, Gerrit REST)
- `src/Macroscope/`: crawler worker orchestration
- `src/Monocle/Prelude.hs`: custom Prelude (re-exports Relude + project utilities)
- `codegen/Monocle/`: protobuf-generated Haskell types (do not edit manually)

Effects use the `effectful` library. The codebase uses `Relude` as its Prelude replacement.

### Frontend (ReScript/React)
- `web/src/App.res`: routing and top-level layout
- `web/src/components/`: UI components (one file per view/feature)
- `web/src/messages/`: protobuf-generated ReScript types (do not edit manually)
- `web/src/components/Prelude.res`: shared utilities and PatternFly wrapper components (`MonoCard`, `MStack`, `MGrid`, `MCenteredContent`, `SortableTable`, etc.)
- `web/src/components/WebApi.res`: generated HTTP client stubs

UI uses PatternFly 4. Component patterns: `MonoCard` for metric cards, `QueryRenderCard` for data-fetching cards with loading states, `SortableTable` for ranked lists.

### API Contract
Defined in `schemas/monocle/protob/*.proto`. The `http.proto` file defines REST routes and is the source of truth for both the Haskell server stubs and the ReScript client stubs. After changing any `.proto` file, run `just codegen`.

### Configuration
`etc/config.yaml` defines workspaces, crawlers, projects, identity aliases, and groups. Schema is in `schemas/monocle/protob/config.proto` and `schemas/config/`. Secrets (tokens, API keys) go in `.secrets` (loaded by docker-compose) or `.env` (loaded by `just`).

## Domain Glossary (ActivePeopleView and Data Model)

This is the most confusing area of the codebase. Here is exactly what each term means, traced to the GitHub API source.

### Change

A **Change** is a GitHub Pull Request. Each PR becomes one `EChange` document in Elasticsearch. Key fields pulled from the GitHub GraphQL API:

- `author` — the GitHub login who opened the PR
- `merged_by` — the GitHub login who clicked Merge
- `state` — `Open`, `Merged`, or `Closed`/`Abandoned`
- `approvals` — array of review decision strings (e.g. `["APPROVED"]`, `["CHANGES_REQUESTED"]`)
- `commit_count`, `additions`, `deletions`, `changed_files_count`
- `created_at`, `merged_at`, `closed_at`

Source: `src/Lentille/GitHub/PullRequests.hs` → `transPR()`, schema in `schemas/monocle/protob/change.proto`

### Review

A **Review** is a GitHub `PullRequestReview` event from the PR timeline. It is **not** a generic comment — it is a formal GitHub review submission that produces one of these states: `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`, `DISMISSED`, or `PENDING`.

Each `PullRequestReview` becomes a `ChangeReviewedEvent` document. The `approval` field stores the review state string (e.g. `"APPROVED"`).

**The "Top reviewers" metric counts unique authors of `ChangeReviewedEvent` documents.** Someone who clicks "Approve" on GitHub is a reviewer. Someone who submits a review with `CHANGES_REQUESTED` or `COMMENTED` state is also counted as a reviewer, because GitHub models all of these as a `PullRequestReview`.

Source: `src/Lentille/GitHub/Utils.hs` → `toMaybeReviewEvent()`, fetched via `timelineItems(types: [PULL_REQUEST_REVIEW])` in the GraphQL query.

### Comment

A **Comment** is a direct PR-level discussion comment — the kind posted in the main conversation thread of a pull request. These come from the `comments` field on the GitHub PR object (not from the timeline).

**Important distinction from Reviews:** GitHub has two separate concepts that Monocle maps differently:

| GitHub concept | GitHub API field | Monocle event | Counted as |
|---|---|---|---|
| Formal review (approve/request changes/review comment) | `timelineItems { ... on PullRequestReview }` | `ChangeReviewedEvent` | Review |
| Direct PR discussion comment | `comments { nodes { ... } }` | `ChangeCommentedEvent` | Comment |
| Inline code comment within a review | Part of `PullRequestReview`, not fetched separately | (not stored) | Review |

So when a user clicks "Start a review", writes inline code comments, and submits — that whole submission counts as one **Review** event. When a user just types in the PR's main comment box and hits "Comment", that counts as one **Comment** event.

Source: `src/Lentille/GitHub/Utils.hs` → `toMaybeCommentEvent()`, `src/Lentille/GitHub/PullRequests.hs` → `getCommentEvents()`

### Author

An **Author** is a GitHub login (username) who performed a specific action. The term is relative to context:

- For a **Change**: the author is whoever opened the PR
- For a **ChangeReviewedEvent**: the author is whoever submitted the review
- For a **ChangeCommentedEvent**: the author is whoever posted the comment

In Elasticsearch, authors are stored as `Ident` objects with `uid` (GitHub login), `muid` (canonical identity, for mapping the same person across multiple systems), and `groups` (team memberships from config). See `schemas/monocle/protob/change.proto` and identity mapping config.

### The 6 Metrics on ActivePeopleView

The page shows 6 ranked-author lists, each backed by a different Elasticsearch query. Here is what each one actually counts, traced to the query type:

| Card title | Query type | What is counted |
|---|---|---|
| By changes created | `Query_top_authors_changes_created` | Authors ranked by number of `ChangeCreatedEvent` documents they authored — i.e. how many PRs they opened |
| By changes merged | `Query_top_authors_changes_merged` | Authors ranked by number of their PRs that reached `state:merged` — same person as change author, filtered to merged state |
| By reviews | `Query_top_authors_changes_reviewed` | Authors ranked by number of `ChangeReviewedEvent` documents they authored — i.e. how many GitHub PullRequestReview submissions they made |
| By comments | `Query_top_authors_changes_commented` | Authors ranked by number of `ChangeCommentedEvent` documents they authored — i.e. how many direct PR discussion comments they posted |
| By changes reviewed | `Query_top_reviewed_authors` | PR authors ranked by how many `ChangeReviewedEvent` documents target their changes — i.e. whose PRs got the most review activity |
| By changes commented | `Query_top_commented_authors` | PR authors ranked by how many `ChangeCommentedEvent` documents target their changes — i.e. whose PRs got the most comments |

The top 3 cards measure **who is doing work** (author of events). The bottom 3 measure **whose work is receiving attention** (the PR author whose change has events on it). Source: `src/Monocle/Backend/Queries.hs`

## Key Patterns

**Adding a new page/route**: Add to `web/src/App.res` nav list and route match, create component in `web/src/components/`.

**Adding a new metric card**: Use `QueryRenderCard` with a `SearchTypes.query_request_query_type` variant and a `match` function to extract the right response variant from `SearchTypes.query_response`.

**Haskell tests**: Live in `test/Spec.hs` using the Tasty framework. Test fixtures (YAML, JSON) in `test/data/`. Tests that hit Elasticsearch require `just elastic` running first.
