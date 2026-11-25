# Workflow: cloud and local

The project uses a mixed flow. Code always reaches `main` through a small PR that Juan reviews and squash-merges.

| Where | Who | Good for | Not for |
| --- | --- | --- | --- |
| Cloud (project thread) | Claude, with GitHub access | Server, simulator, docs, CI, PRs, reviews. Works while Juan is away | Anything that needs Unity, Docker or .NET installed |
| Local session (Remote Control, or Claude Code on Juan's PC) | Claude Code, with Juan's tools | Creating the Unity project, EditMode/PlayMode tests, reading the Unity console, running the API against Docker | Long unattended work: it only runs while the PC is on |
| Juan | Decides and verifies | Approves each local session, runs the PR test steps, merges | |

## Rules
1. **Cloud first.** If a story can be built and reviewed without Unity, it is done in the cloud and verified by Juan with the PR's test steps.
2. **Local only when needed.** Each local session is requested for one purpose (for example "create the Unity project for K-01") and approved by Juan first.
3. **Everything goes through a PR**, including what a local session produces: it works on a branch and never pushes to `main`.
4. **Report what ran.** A PR that was not executed says so in its notes.

## Tools on Juan's PC
- Unity Hub + Unity 6.6 (6000.6.10f1) with Android Build Support.
- Unity CLI (`unity`), plus the official Unity plugin for Claude Code once the Unity project exists:
  `unity pipeline install` and `unity mcp configure claude` inside `unity-client/`, then
  `/plugin marketplace add Unity-Technologies/unity-agent-plugin` and `/plugin install unity@unity-agent-plugin`.
- Docker Desktop (WSL 2), .NET SDK 10.0.404, Git + Git LFS.

## Which story uses which
| Stories | Where |
| --- | --- |
| K-01 Unity base project | Local (create project) + cloud (PR with structure) |
| K-02, K-03 Collection and battle player | Cloud code, local test run |
| K-04 OpenAPI contract | Cloud |
| K-05 IGameApi | Cloud code, local PlayMode run |
| K-06, K-07 Simulator, data model | Cloud (tests run in CI) |
| K-08 to K-12 Backend | Cloud (tests run in CI) |
| K-13 End-to-end integration | Local |
| K-14, K-15, K-16 Config, deploy, portfolio | Cloud |
