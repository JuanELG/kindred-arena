# Kindred Arena: architecture and decisions

Last reviewed: 2026-11-24. Source of truth for the diagrams and screens: the design artifacts linked in [`docs/README.md`](README.md).

## Summary

Kindred Arena is a thin Unity client on top of an ASP.NET Core API that is the single source of truth: it stores player state, simulates battles and applies the economy. Everything runs as a modular monolith deployed as one container with managed PostgreSQL.

Principles:

1. **Server-authoritative.** The client sends intentions; the server decides outcomes. Gold, rarity rolls and battle results never come from the client.
2. **Determinism.** A battle is a pure function `(teamA, teamB, seed) -> log`, shared by server and Unity and covered by golden tests.
3. **Modular monolith first.** One service with clear modules (Auth, Players, Economy, Battles, Seasons, Config). It can be split later if needed, not now.
4. **Configurable without redeploy.** Prices, drop rates, rewards and events live in versioned remote configuration.
5. **Zero cost.** Everything fits in free tiers; no paid services in the MVP.

## Context and containers (C4 levels 1 and 2)

The game never talks to the database or to any other service: everything goes through the API. An admin panel (Blazor, phase 2) will use the same API with an admin role.

In Unity, features only know `IGameApi`, so the mock and the real HTTP client are interchangeable. In the API, `Application` orchestrates each use case inside a transaction and `Domain` holds the rules without knowing EF Core. The battle simulation is the same DLL on both sides. The diagrams live in the design artifact (not exported here).

## Code structure

```
kindred-arena/
  server/
    src/
      Kindred.Api/             # Endpoints (Minimal APIs), auth, middleware, DI
      Kindred.Application/     # Use cases: PurchaseEgg, StartBattle, ClaimIdle...
      Kindred.Domain/          # Entities and rules: Player, OwnedCreature, Wallet
      Kindred.Infrastructure/  # EF Core, PostgreSQL, repositories, migrations
    tests/
      Kindred.UnitTests/       # Domain and use cases (xUnit)
      Kindred.IntegrationTests/# API + real Postgres with Testcontainers
  shared/
    Kindred.Simulation/        # netstandard2.1: battle simulator + log DTOs
    Kindred.Simulation.Tests/  # Golden battles with a fixed seed
  unity-client/
    Assets/_Project/
      Scripts/Core/            # Bootstrap, composition, events
      Scripts/Api/             # IGameApi, HttpGameApi, MockGameApi, DTOs
      Scripts/Features/        # Collection, Team, Hatchery, Arena, Season
      Data/Creatures/          # CreatureDefinition (ScriptableObject) x12
      Tests/EditMode/ and Tests/PlayMode/
  docs/                        # Architecture, ADRs, backlog
  docker-compose.yml           # Local development database (API added later)
  .github/workflows/           # CI: build, tests, Docker image
```

Dependency rules: `Api -> Application -> Domain`; `Infrastructure` implements interfaces from `Application`. `Domain` knows neither EF Core nor HTTP. In Unity, features depend on `IGameApi`, never on `UnityWebRequest` directly.

How `Kindred.Simulation` reaches Unity: it is built as a `netstandard2.1` DLL and copied to `Assets/Plugins/` by a build script, or published as a local UPM package. We start with the copy because it is simplest.

## ADRs

Each ADR answers an interview question: what we chose, why, and what it costs. Status of all: accepted (Nov 2026), except ADR-009.

### ADR-001: own backend on ASP.NET Core
- **Context:** the goal is to show .NET backend skills, not to integrate another SDK. PlayFab and Firebase are already in the CV.
- **Decision:** own API on ASP.NET Core (.NET 10, LTS). Updated 2026-11-24: .NET 8 left support on 10 Nov 2026.
- **Rejected:** Nakama (fast, but Go/TS and little .NET), PlayFab or Unity Gaming Services (repeats what the CV already shows).
- **Consequences:** more own work (auth, migrations, deploy), but every piece can be explained and defended.

### ADR-002: modular monolith
- **Context:** one developer, about 4 hours a week, no real traffic.
- **Decision:** one service with modules separated by folder and namespace; each module owns its tables.
- **Rejected:** microservices (operational complexity with no benefit at this scale).
- **Consequences:** one deployment, one database. If a module grows (for example Battles), its boundaries are already drawn.

### ADR-003: REST + JSON for the client
- **Context:** Unity needs a simple, debuggable client; calls are low frequency.
- **Decision:** versioned REST (`/api/v1`) with OpenAPI; errors as `ProblemDetails`. The OpenAPI document comes from the built-in `Microsoft.AspNetCore.OpenApi` generator, with Scalar as the free UI instead of Swashbuckle.
- **Rejected:** gRPC (more fragile Unity support, especially on mobile and WebGL). Real time is left for SignalR after the MVP.
- **Consequences:** easy to test from the API UI and to mock in Unity.

### ADR-004: PostgreSQL + EF Core
- **Context:** the economy needs transactions and integrity between gold, inventory and purchases.
- **Decision:** PostgreSQL with EF Core and migrations; `jsonb` for remote config and the battle log.
- **Rejected:** MongoDB (no comfortable cross-collection transactions for this case), Redis as the main database.
- **Consequences:** clear relational model; Redis only if a real caching need appears.

### ADR-005: server-simulated, deterministic battles
- **Context:** if the client decides the winner, anyone can cheat in the ranking.
- **Decision:** the server generates the `seed`, runs the simulation and stores the result. The simulator lives in `Kindred.Simulation` (netstandard2.1), without `System.Random` or `float`: own PRNG and integers.
- **Consequences:** free replays (seed + teams), reproducible golden tests and the same code in Unity for previews.

### ADR-006: anonymous device login + JWT
- **Context:** a casual mobile game should not ask for sign-up to start.
- **Decision:** `POST /auth/device` creates or recovers the player; 15-minute JWT access token and a 30-day rotating refresh token stored as a hash.
- **Consequences:** losing the device means losing the account. Linking an email or Google account is out of the MVP.

### ADR-007: idempotency and optimistic concurrency in the economy
- **Context:** on mobile, retries over flaky networks are normal; a double tap must not charge twice.
- **Decision:** mandatory `Idempotency-Key` header on economic commands (the stored response is returned on retry) and a `RowVersion` on `Player` to detect concurrent writes.
- **Consequences:** one more table and periodic cleanup of old keys (for example after 24 hours).

### ADR-008: own versioned remote config
- **Context:** prices, probabilities and events change without shipping a new game version.
- **Decision:** a `ConfigVersion` table with schema-validated JSON; one active version; `GET /config` with ETag. The server uses the same config to charge, so client and server never differ.
- **Rejected:** Firebase Remote Config (the server also needs the values, and it would be another SDK).
- **Consequences:** rollback means activating the previous version. Later the Blazor panel edits it.

### ADR-009: container deploy on a free tier (proposed)
- **Context:** zero cost and something that can be shown live.
- **Proposed decision:** Docker image built in GitHub Actions, deployed to a PaaS free tier with free managed PostgreSQL.
- **Pending:** pick the provider on 5 January, because free tiers change often; check which one is still valid that day.

### ADR-010: async PvP in the MVP
- **Context:** real time (netcode, rooms, latency) multiplies scope.
- **Decision:** the player fights another player's saved team, chosen by nearby rating.
- **Consequences:** no dedicated game servers. Real-time battles with SignalR remain phase 2.

## Quality attributes

Targets are modest and measurable, suited to a free-tier portfolio. The numbers are goals, not measurements.

| Attribute | MVP goal | How |
| --- | --- | --- |
| Security | No economy or ranking cheats | Server-authoritative, short JWT, hashed rotating refresh token, input validation, secrets in environment variables |
| Abuse | One player cannot saturate the API | Per-player rate limiting on `/battles` and `/shop/purchase` (built-in ASP.NET Core middleware) |
| Consistency | Never negative gold or double purchases | Transactions, `RowVersion`, `Idempotency-Key` |
| Performance | p95 under 200 ms on read endpoints | Indexes on `PlayerId` and `Rating`, 30 s leaderboard cache; validated with k6 |
| Observability | Know what failed and for whom | Structured logs with Serilog, correlation id per request, `/health` checks |
| Quality | Safe changes every Tuesday | GitHub Actions CI: build, unit tests, Testcontainers integration tests and simulator golden tests |
| Deployment | One command or one push | Docker image; migrations on startup in development and as a separate step in production |
| Cost | 0 USD per month | Free tiers; no paid services (ADR-009) |

**Out of the MVP on purpose:** distributed cache, message queues, multiple regions, autoscaling. Mentioning them in interviews as "next step if there were real load" earns more than building them without need.
