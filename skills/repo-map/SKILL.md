---
name: repo-map
description: >
  Walk a code repository and produce an architecture overview: business purpose, tech stack,
  a component diagram (plain-text ASCII boxes, readable in raw markdown), and per-component
  explanations of what each piece does and how it relates to others. Use when the user wants
  to understand, map, or onboard onto an
  unfamiliar codebase, asks "explain this repo's architecture", "how are things wired together
  here", "give me the big picture", or invokes `/repo-map`.
---

# Repo Map

Produce a top-down architecture explanation of a repo: overview → business logic → tech stack →
diagram → component breakdown → relationships. Order matters — give the reader the big picture
before the details.

Not a code review (see `code-review`/`review` skills) and not a reading-order for a diff (see
`mr-reading-order`). This is for understanding a whole repo's shape, cold.

## Steps

### 1. Scope the walk

Default to the whole repo. If the user names a subdirectory, service, or package, scope to that
instead. Confirm scope only if genuinely ambiguous (e.g. a monorepo with 10 unrelated services and
no hint which one).

### 2. Gather signal before writing anything

Prefer delegating this to an `Explore` or `general-purpose` agent if the repo is large — this step
is read-heavy and shouldn't consume the main thread's context budget. Collect:

- **Manifest/build files**: `package.json`, `go.mod`, `build.gradle*`, `pom.xml`, `pyproject.toml`,
  `Cargo.toml`, `Gemfile`, `*.csproj` — language, framework, key deps, scripts.
- **Entry points**: main/index files, `cmd/`, server bootstrap, CLI entry, Lambda handlers.
- **Top-level structure**: `ls`/tree of the repo root and one level into major dirs. Note whether
  it's organized by feature/domain or by technical layer.
- **Config & infra**: Dockerfiles, k8s manifests, `docker-compose.yml`, CI pipelines
  (`.gitlab-ci.yml`, `.github/workflows/`), IaC — these reveal deployment topology and external deps
  (databases, queues, caches) even when not obvious from application code.
- **Existing docs**: root `README.md`, `CLAUDE.md`, `docs/`, `ARCHITECTURE.md` — read these first,
  they often already state intent that would otherwise need inference.
- **Data layer**: migrations, schema files, ORM models — tells you the core domain entities.
- **Boundaries**: HTTP route definitions, gRPC/proto files, message queue consumers/producers,
  webhook handlers — these are the component-to-component and component-to-external edges.
- **Tests**: skim test directory names/structure — often the clearest map of intended component
  boundaries, since tests are usually organized around units of behavior.

For each major component found, identify: its responsibility, what calls it, what it calls,
and what external systems (DB, queue, third-party API) it touches.

### 3. Determine business logic, not just structure

Don't stop at "this is a Kotlin Spring Boot service with a controller/service/repo layout."
Answer: what real-world process does this code exist to run? Look at domain names (entities,
use-case names, event names), not just framework layers, to infer this. E.g. "this service
manages beneficiary onboarding and KYC state transitions," not "this is a CRUD API."

### 4. Write the overview first

Lead with 3-5 sentences answering: what does this system do, for whom, and why does it exist.
No headers needed for something this short. This must come before any diagram or component list.

### 5. Tech stack section

Short list: language(s), primary framework, datastore(s), messaging/queue, notable libraries that
shape architecture (not every dependency — only ones that matter to how the system is built:
ORM, DI framework, API framework, testing framework if notable).

### 6. Component diagram

Default to a plain-text ASCII box-and-arrow diagram inside a fenced code block (no mermaid) —
readable directly in raw markdown/terminal without a renderer. Boxes (`┌─┐│└─┘`) for
components/services/layers, arrows (`▼ ▶ ◀ ▲` or `-->`, `│`) for calls/data-flow, and a short
label next to or under each arrow naming the relationship (e.g. `RPC`, `REST call`, `publishes
event`, `reads/writes`) — don't leave edges unlabeled. Keep to components that matter for
understanding the system — don't diagram every file, diagram every meaningfully distinct
responsibility. Group cross-cutting concerns (cache, feature flags, cross-DC routing) in their
own labeled cluster at the bottom rather than cluttering the main flow.

If the repo is a single monolith without clear service boundaries, diagram internal layers/modules
instead (e.g. API layer → domain/service layer → repository layer → DB), still with labeled edges.

Only fall back to a mermaid diagram if the user explicitly asks for mermaid, or the shape is
genuinely graph-like (many-to-many edges) rather than layered/hierarchical, where ASCII boxes
would become unreadable.

### 7. Per-component breakdown

For each node in the diagram, a short paragraph or bullet:
- What it's responsible for (business terms, not "handles requests")
- Key files/dirs backing it (`path/to/thing` — so the user can navigate there)
- What it depends on and what depends on it (ties back to the diagram edges)
- Anything notable/unusual about how it's implemented, if relevant

Order this section to match a sensible reading order (e.g. entry point → core domain → persistence
→ integrations), not alphabetical or diagram-node order.

### 8. Keep it navigable

Use `file_path:line_number` references so the user can jump straight to source. Don't paste large
code blocks — point to where things live and summarize.

## Output shape

1. Overview (prose, no headers)
2. Business logic (prose, no headers)
3. Tech stack (short list)
4. Architecture diagram (ASCII box-and-arrow, plain text)
5. Component breakdown (one section per component, reading order)

Don't add a "next steps" or summary section unless the user asks — the breakdown is the deliverable.
