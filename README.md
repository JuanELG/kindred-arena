# kindred-arena
Monster-collector mobile game with server-authoritative async PvP. Unity client + ASP.NET Core (.NET 10) + PostgreSQL, deterministic battle simulation.

> Work in progress: MVP target is 12 Jan 2027. See `docs/` for architecture and backlog.

## Repository layout

| Folder | What lives there |
| --- | --- |
| `server/` | ASP.NET Core API (Minimal APIs), Application/Domain/Infrastructure, tests |
| `shared/` | `Kindred.Simulation` (netstandard2.1): deterministic battle simulator used by server and Unity |
| `unity-client/` | Unity 6.6 (6000.6.10f1) project, URP mobile |
| `docs/` | Architecture, ADRs, backlog |

## Prerequisites
- .NET SDK 10.0.404 (pinned in `global.json`)
- Docker Desktop (WSL 2 backend)
- Git + Git LFS (`git lfs install`)
- Unity 6.6 (6000.6.10f1) with Android Build Support

## Local database
```bash
cp .env.example .env      # PowerShell: Copy-Item .env.example .env
docker compose up -d db
docker compose ps         # db should be "healthy"
```
Stop it with `docker compose down`. Data stays in the `kindred-pgdata` volume.

## License
MIT
