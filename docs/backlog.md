# MVP backlog

The MVP is 16 stories spread over 8 Tuesday sessions of 3 to 4 hours, closing on 2027-01-12.

## How it is used
Each Tuesday takes the stories of that row, closes them against their DoD and updates the status. If a story does not fit, it is split in two; it is never carried over half done.

Story format: *As a [player or operator], I want [action] so that [benefit].* Acceptance criteria use Given / When / Then and each one becomes a test.

**Definition of Ready (before starting)**
- The story has verifiable acceptance criteria.
- It fits in one Tuesday (4 hours or less); if not, it is split.
- Its dependencies are Done.
- It is clear which endpoint or screen it touches.

**Definition of Done (to close)**
- Every acceptance criterion has a test and passes in CI.
- Small PR (under 300 lines) merged to `main`.
- No new warnings; errors returned as `ProblemDetails` where it applies.
- OpenAPI or README updated if a contract changed.
- A one-line note in the Tuesday log: what was learned.

## Board

| Tuesday | ID | Story | Epic | Hours |
| --- | --- | --- | --- | --- |
| 24 Nov | K-01 | Unity base project and structure | Client | 2 |
| 24 Nov | K-02 | Creature catalog and local Collection | Client | 1 |
| 24 Nov | K-03 | Battle player from a JSON log | Client | 1 |
| 1 Dec | K-04 | MVP OpenAPI contract | Contract | 1.5 |
| 1 Dec | K-05 | IGameApi with mock and HTTP client | Client | 2 |
| 8 Dec | K-06 | Deterministic simulator with golden tests | Battle | 2.5 |
| 8 Dec | K-07 | Data model and migrations | Data | 1.5 |
| 15 Dec | K-08 | Device login with JWT | Backend | 1 |
| 15 Dec | K-09 | Player profile and team | Backend | 1 |
| 15 Dec | K-10 | Battles with Elo and season leaderboard | Backend | 2 |
| 22 Dec | K-11 | Idempotent egg shop | Economy | 3.5 |
| 29 Dec | K-12 | Level up and idle gold | Economy | 1.5 |
| 29 Dec | K-13 | End-to-end Unity integration | Client | 2 |
| 5 Jan | K-14 | Versioned remote config | Live ops | 2 |
| 5 Jan | K-15 | Deploy, CI/CD and load test | DevOps | 2 |
| 12 Jan | K-16 | README, video and case study | Portfolio | 3.5 |

15 December is the heaviest Tuesday: if it slips, K-10 moves to the 22nd and K-11 is split into purchase and egg opening.

## Stories and acceptance criteria

### 24 Nov: client base

**K-01 Unity base project.** As a developer, I want a clear structure so every feature grows without coupling.
- [ ] Given the repo, when I open `unity-client/`, then `Assets/_Project/` exists with `Core`, `Api`, `Features`, `Data` and `Tests`, each with its assembly definition.
- [ ] Given the project, when I run the EditMode tests, then at least one test passes in CI.
- [ ] Given the project, when I search for static singletons, then there are none: composition happens in a `Bootstrap`.

**K-02 Local Collection.** As a player, I want to see my creatures to know what I have.
- [ ] Given 12 `CreatureDefinition` (ScriptableObject), when I open Collection, then I see the owned ones with name, element and rarity, and the rest locked.
- [ ] When I tap an owned creature, then I see its level, HP, Attack and Speed.

**K-03 Battle player.** As a player, I want to watch the battle animated to understand why I won or lost.
- [ ] Given a sample JSON log, when I load it in Arena, then each event (`attack`, `damage`, `faint`, `end`) plays in order.
- [ ] Given a log with an unknown event, when it plays, then it is ignored with a warning and nothing breaks.

### 1 Dec: contract and client

**K-04 OpenAPI contract.** As a developer, I want the contract defined before the code so client and server do not diverge.
- [ ] `docs/openapi.yaml` exists with the 13 MVP endpoints, their requests, responses and errors.
- [ ] Every error has a stable code (`insufficient_gold`, `invalid_team`, `rate_limited`...).

**K-05 IGameApi.** As a developer, I want an API interface so I can work in Unity without a server.
- [ ] Given `MockGameApi`, when I use it in Play Mode, then every screen works offline.
- [ ] Given `HttpGameApi`, when the network fails, then it retries with backoff and then shows an error, without freezing the UI.
- [ ] Calls are `async` and accept a `CancellationToken`.

### 8 Dec: battle and data

**K-06 Deterministic simulator.** As a player, I want fair, reproducible battles.
- [ ] Given the same teams and seed, when I simulate twice, then the log is byte-for-byte identical.
- [ ] At least 4 golden battles (element advantage, speed tie, 30-turn cap, perfect victory) pass in CI.
- [ ] The simulator does not use `System.Random`, `float` or the clock.

**K-07 Data model.** As a developer, I want a versioned schema so I can evolve it without losing data.
- [ ] The entities in the design have their EF Core migration and apply on an empty Postgres with Docker Compose.
- [ ] The 12 creatures are seeded by migration.

### 15 Dec: core API (on the .NET project base)

**K-08 Device login.** As a player, I want to get in without signing up.
- [ ] Given a new device id, when I call `POST /auth/device`, then the player is created with starting gold and I receive access and refresh tokens.
- [ ] Given an already-used refresh token, when I reuse it, then I get 401 and the chain is revoked.

**K-09 Profile and team.** As a player, I want to save my team so others can fight it.
- [ ] Given a team with a creature I do not own or a repeated one, when I call `PUT /team`, then I get 400 `invalid_team`.
- [ ] `GET /players/me` returns current gold, rating and team.

**K-10 Battles and leaderboard.** As a player, I want to fight and climb the season ranking.
- [ ] When I call `POST /battles`, then the server picks an opponent with a close rating, simulates, stores seed and result, and applies Elo and reward in one transaction.
- [ ] `GET /battles/{id}` returns the same log as the original battle.
- [ ] `GET /leaderboards/season/current` returns the top 100 and my position.

### 22 Dec: shop

**K-11 Idempotent egg shop.** As a player, I want to open eggs without fear of being charged twice.
- [ ] Given enough gold, when I buy, then gold is deducted and I get a creature from the server drop table.
- [ ] Given the same `Idempotency-Key`, when I repeat the purchase, then I get the same response and gold is deducted once.
- [ ] Given insufficient gold, when I buy, then I get 409 `insufficient_gold` and nothing changes.
- [ ] Given a duplicate creature, then it converts into gold.

### 29 Dec: progress and integration

**K-12 Level up and idle.** As a player, I want to improve my creatures and collect what they generated without playing.
- [ ] Leveling up costs `50 x current level` in one transaction.
- [ ] Given I change the phone clock, when I claim idle, then the amount does not change: it uses server time, capped at 8 hours.

**K-13 End-to-end integration.** As a player, I want the game to use the real server.
- [ ] With `HttpGameApi` against local Docker the loop completes: login, idle, egg, team, battle and ranking.
- [ ] A PlayMode test walks that loop against the mock.

### 5 Jan: live ops and production

**K-14 Remote config.** As an operator, I want to change prices and events without shipping another version.
- [ ] Given I activate a new config version, when the client calls `GET /config`, then it receives the new values and the server charges with them.
- [ ] Given a config that is invalid by schema, then it is rejected and the active version does not change.
- [ ] Rolling back to the previous version is a single operation.

**K-15 Deploy.** As an operator, I want the API public and stable.
- [ ] A push to `main` with green tests builds the image and deploys it.
- [ ] `/health` returns 200 in production.
- [ ] A k6 read test records its p95 in the README.

### 12 Jan: portfolio

**K-16 README, video and case study.** As a candidate, I want to show the project in 2 minutes.
- [ ] README with architecture diagram, how to run it and a link to the ADRs.
- [ ] 90-second video of the full loop with the API in production.
- [ ] 3 CV bullets and a LinkedIn post.

## Out of the MVP

None of these enters before 12 January.

| ID | Story | Why later |
| --- | --- | --- |
| K-17 | Real-time battles with SignalR | Phase 2: adds netcode and rooms; pushes toward MMO and multiplayer |
| K-18 | Admin panel in Blazor | Phase 2: edits config and shows KPIs on the same API |
| K-19 | Link account with email or Google | Needed for real production, not for the portfolio |
| K-20 | KPI dashboard (retention, gold in circulation, win rate) | Depends on having traffic and the panel |
| K-21 | Art direction and juice on the screens | Gameplay and visual life are picked up after the MVP |
