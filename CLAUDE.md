# Kindred Arena: instructions for Claude Code

Monster-collector mobile game with server-authoritative async PvP. Unity client + ASP.NET Core (.NET 10) + PostgreSQL. The battle is a deterministic function `(teamA, teamB, seed) -> log` shared by server and client.

Read first: `docs/architecture.md` (ADRs), `docs/backlog.md` (stories K-01..K-16) and `docs/workflow.md` (who does what).

## Language
- Talk to Juan in Spanish. Code, comments, commit messages, PR text and docs in this repo are in English.

## Layout
- `server/`: ASP.NET Core API (Minimal APIs). `Api -> Application -> Domain`; `Infrastructure` implements `Application` interfaces.
- `shared/Kindred.Simulation/`: `netstandard2.1`, no `System.Random`, no `float`, no clock. Own PRNG, integers only. Golden tests with fixed seeds.
- `unity-client/`: Unity 6.6 (6000.6.10f1), URP mobile, IL2CPP/ARM64. Code under `Assets/_Project/`.
- `docs/`: architecture, ADRs, backlog.

## Conventions
- .NET SDK is pinned by `global.json` (10.0.404). Do not bump it without a PR that says why.
- No static singletons. Compose dependencies in a `Bootstrap` (Unity) or DI (server).
- Unity features depend on `IGameApi`, never on `UnityWebRequest` directly. Calls are `async` and take a `CancellationToken`.
- Prefer ScriptableObjects for data, composition over inheritance, guard clauses.
- Every acceptance criterion in `docs/backlog.md` becomes a test (Arrange-Act-Assert). Unity: EditMode and PlayMode. Server: xUnit.
- Never commit secrets. `.env` is local only (`.env.example` is the template).
- Binary art and audio go through Git LFS (see `.gitattributes`).

## Git and PRs
- Never push to `main`. One branch per story, named like `feat/k-01-unity-base`.
- Small PRs (under 300 lines), squash-merged by Juan. PR description: what, how to test, notes.
- Commit only what the story needs. Do not commit `Library/`, `Temp/`, `Logs/` or generated `.csproj` files from Unity.

## When running locally (Remote Control or Claude Code on Juan's PC)
- You are here for work that needs the Unity editor, Unity CLI, Docker or .NET: creating the Unity project, running EditMode/PlayMode tests, checking the console, validating end-to-end integration.
- Do not invent Unity CLI flags: run `unity --help` and read the output first.
- Report exactly what you ran and what it printed. If a test or build fails, say so with the output; never skip or disable tests to get green.
- Do not start long-running builds without saying so first.
