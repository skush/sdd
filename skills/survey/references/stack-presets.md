# Greenfield stack presets — Node.js and .NET backends

> **Reference-only.** Read by [`foundation.md`](./foundation.md) step G4. A preset is a complete,
> coherent foundation bundle for a backend: stack, layout, persistence, conventions and the
> machine keys `docs/architecture-map.md` needs. Picking a preset replaces the stack half of G4 with
> one question; everything in it can still be overridden piece by piece afterwards.

## How G4 uses the presets

1. **Hard constraint first.** If G3 surfaced a hard constraint («must be Python», «must run on
   the JVM»), skip the presets and run G4 as before — never push a preset against a stated constraint.
2. **Otherwise ask ONE question, first in G4**, phrased per
   [`../../_shared/ask-style.md`](../../_shared/ask-style.md), with these options in this order:
   - **«Node.js backend (Recommended)»** — the Node preset below.
   - **«.NET backend»** — the .NET preset below.
   - **«Something else»** — fall back to the generic G4 menu in [`foundation.md`](./foundation.md).
   Gloss what each preset brings (framework, database, test tools) in the description, so a junior
   can choose without knowing the names.
3. **Then adapt to the intent.** No datastore needed (a CLI, a stateless proxy) → drop the
   database, the migration tool and scaffold task S3 (with the reason stated). A frontend → keep
   the backend preset and pick the frontend separately; presets cover the backend only.
4. **Confirm as one bundle** at *guided-default* depth; walk the pieces at *guided-explained* /
   *expert* depth. Every override is recorded; the preset is a starting point, not a lock.
5. The chosen preset's **machine keys** go straight into the map's frontmatter in G5, and its
   **foundational ADRs** are the usual three (stack, module style, persistence), each naming the
   preset as the option chosen and the other preset as an alternative considered.

## Preset: Node.js backend

| Decision | Pick | Why (one line, for the description) |
|---|---|---|
| Language / runtime | TypeScript on Node.js (current LTS) | types catch mistakes before runtime; LTS gets security fixes for years |
| Web framework | Fastify | fast, small, schema-based request validation built in |
| Datastore | PostgreSQL | the default relational database; JSON support when needed |
| DB access | `pg` + Kysely (type-safe SQL query builder) | plain SQL you can read, with TypeScript checking the column names |
| Migrations | `node-pg-migrate` | plain up/down migration files, so the smoke test can apply **and** revert |
| Tests | Vitest (unit) + Testcontainers (integration against a real Postgres in Docker) | fast unit runs; integration tests use the real database, not a mock |
| Lint / format | ESLint (`typescript-eslint`) + Prettier | one style, enforced in CI |
| Style | modular: `src/modules/<module>/{domain,app,infra,http}` | each feature module owns its layers; fits SDD's per-feature flow |

Layout:

```
src/
  main.ts                  # boots Fastify, registers modules
  modules/<module>/...
  shared/                  # error envelope, config, db pool
migrations/
test/                      # integration tests (unit tests sit next to the code as *.test.ts)
docker-compose.yml         # local Postgres
AGENTS.md
```

Machine keys for the map:

```yaml
language: "typescript (node lts)"
build_cmd: "npm run build"            # tsc -p .
test_cmd: "npm test"                  # vitest run
lint_cmd: "npm run lint"              # eslint . && prettier --check .
migration_tool: "node-pg-migrate"     # npm run migrate up / npm run migrate down
frontend: ""
```

## Preset: .NET backend

| Decision | Pick | Why (one line, for the description) |
|---|---|---|
| Language / runtime | C# on .NET (current LTS) | strongly typed, fast, long support window |
| Web framework | ASP.NET Core minimal APIs | little ceremony; endpoints are plain functions |
| Datastore | PostgreSQL (SQL Server if the user's environment requires it) | same reasoning as the Node preset |
| DB access | EF Core with the Npgsql provider | the standard .NET ORM; LINQ queries checked by the compiler |
| Migrations | EF Core migrations (`dotnet ef`) | generated from the model; revert = update to the previous migration |
| Tests | xUnit + `WebApplicationFactory` + Testcontainers for .NET | in-memory test server for HTTP tests, a real Postgres in Docker for integration |
| Lint / format | `dotnet format` + analyzers with warnings as errors | built into the SDK, no extra tools |
| Style | modular folders in one API project: `Modules/<Module>/{Domain,Application,Infrastructure,Endpoints}` | one project per layer is overkill at the start; folders keep the layers visible |

Layout:

```
<App>.sln
src/<App>.Api/
  Program.cs               # builds the app, maps module endpoints
  Modules/<Module>/...
  Shared/                  # error envelope (ProblemDetails), config, DbContext
  Migrations/
tests/<App>.Tests/         # unit + integration
docker-compose.yml         # local Postgres
AGENTS.md
```

Machine keys for the map:

```yaml
language: "c# (.net lts)"
build_cmd: "dotnet build"
test_cmd: "dotnet test"
lint_cmd: "dotnet format --verify-no-changes"
migration_tool: "ef core migrations"  # dotnet ef database update / dotnet ef database update <previous>
frontend: ""
```

## What both presets share

- **Error envelope:** one JSON error shape for every endpoint (RFC 9457 Problem Details).
- **IDs:** app-generated, time-sortable (UUIDv7), per `data-model`'s baseline.
- **CI:** one GitHub Actions workflow running build + lint + test, with a Postgres service for
  integration tests.
- **Agent instructions:** scaffold task S5 writes `AGENTS.md` with the preset's commands and
  conventions (see [`foundation.md`](./foundation.md) G6).
- **Versions:** pin to whatever is the current LTS when `scaffold` runs, and record the exact
  versions in the stack ADR — never copy a version number from this file.
