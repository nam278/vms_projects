# VMS — GPU-Accelerated Video Management System

> Multi-repo workspace: DeepStream analytics engine, Python control-plane backend, and TypeScript desktop/mobile frontend.

## Project

This workspace contains three independent sub-projects, each its own git repository:

| Repo | Stack | Role |
|------|-------|------|
| `vms-engine/` | C++17 · DeepStream 8.0 · CMake · Docker | GPU video analytics: RTSP ingest, AI inference, tracking, smart recording, RTSP/Kafka/Redis output |
| `vms_app_backend/` | Python 3.11 · FastAPI · uv · SQLAlchemy | Control plane: camera topology, event processing, face recognition, camera control, worker orchestration |
| `vms_app_frontend/` | TypeScript · Electron + Vite · Expo RN · pnpm | User-facing clients: desktop monitoring app, mobile/web companion |

Each sub-project has its own **`AGENTS.md`** — read it before working in that repo. The root `reasonix.toml` loads skills from all three.

## Architecture

```
                    ┌──────────────────────┐
RTSP cameras ──────▶│    vms-engine (C++)   │────▶ Redis Streams / Kafka events
                    │  DeepStream pipeline  │────▶ RTSP outputs
                    └──────────┬───────────┘
                               │ events (Redis/Kafka)
                               ▼
                    ┌──────────────────────┐
                    │ vms_app_backend (Py)  │
                    │ FastAPI · DDD · Hex   │────▶ PostgreSQL
                    │ vms_app / face_rec /  │
                    │ camera_control / worker│
                    └──────────┬───────────┘
                               │ REST API
                               ▼
                    ┌──────────────────────┐
                    │vms_app_frontend (TS)  │
                    │ Electron desktop app  │
                    │ Expo mobile app       │
                    └──────────────────────┘
```

- **vms-engine**: ingests RTSP streams, runs TensorRT inference + object tracking, emits detection events to Redis/Kafka and RTSP outputs. Config-driven via YAML. Runs in Docker on NVIDIA GPU.
- **vms_app_backend**: consumes engine events, manages camera topology, event rules, face recognition, and worker lifecycle. DDD + Hexagonal Architecture.
  - **Active code path**: `apps_v2/` — mỗi sub-project là 1 project con riêng biệt:
    - `apps_v2/vms_app/` — main control plane app (FastAPI, eventing, topology, timekeeping)
    - `apps_v2/face_recognition/` — face recognition service (inference, search, register)
    - `apps_v2/camera_control/` — camera control service (Hikvision, ONVIF, PTZ, audio)
    - `apps_v2/worker/` — worker execution plane (manifest, recovery, reconcile)
  - **Canonical docs**: `docs_v2/` — product truth, domain docs, phase execution, engineering docs
  - `apps/` là **legacy** — không dùng nữa
- **vms_app_frontend**: desktop (Electron + TanStack Router) and mobile (Expo Router + Zustand) clients consuming the backend REST API.

## Commands

There is no single build/run command for the whole workspace. Each repo is independent.

### vms-engine (C++ · Docker)

```bash
cd vms-engine
docker build -t vms-engine-dev:latest .
docker compose up -d app
docker compose exec app bash
# Inside container:
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug -DDEEPSTREAM_DIR=/opt/nvidia/deepstream/deepstream
cmake --build build -- -j5
./build/bin/vms_engine -c dev/configs/my_test.yml
```

### vms_app_backend (Python · uv)

```bash
cd vms_app_backend
make install                    # root tooling
cd apps_v2/vms_app && uv sync  # per-app setup
cd apps_v2/vms_app && make format && make typecheck && make test
```

### vms_app_frontend (TypeScript · pnpm)

```bash
cd vms_app_frontend
pnpm install
pnpm dev                        # desktop app
cd apps/mobile && pnpm start    # mobile Metro
```

## Conventions

### Scope and authority
- **Always read the sub-project AGENTS.md first** — it is authoritative for its repo. This root file is only an index.
- Work in the **narrowest possible scope**. Don't spread changes across repos unless the task explicitly requires it.
- When guidance conflicts, narrower scope wins: app-level `AGENTS.md` > repo-level `AGENTS.md` > this workspace root `AGENTS.md`.

### Repo independence
- Each sub-project is an **independent git repo** with its own `.git/`, dependencies, and CI. Changes in one repo don't automatically affect others.
- Cross-cutting contract changes (API schemas, event formats, Redis/Kafka message shapes) should be coordinated but committed separately per repo.

### CodeGraph (code intelligence)

CodeGraph is initialized **per sub-project** (not at workspace root — root `.gitignore` excludes all three sub-project dirs). Each sub-project has its own `.codegraph/` with a local SQLite index.

| Sub-project | Command | Scope |
|---|---|---|
| `vms_app_backend` | `cd vms_app_backend && codegraph init -i` | Python · FastAPI · DDD |
| `vms_app_frontend` | `cd vms_app_frontend && codegraph init -i` | TypeScript · Electron · Expo |
| `vms-engine` | `cd vms-engine && codegraph init -i` | C++ · DeepStream |

When using CodeGraph MCP tools, always pass `projectPath` to target the correct sub-project:

```
codegraph_search(query="SymbolName", projectPath="/home/vms2/Vms/vms_app_backend")
```

To re-index after code changes: `codegraph sync` (from within the sub-project dir).

### Skills (agent knowledge)

Each sub-project has domain-specific skills in its own `.agents/skills/` directory. The workspace root aggregates them via **symlinks**:

```
/home/vms2/Vms/.agents/skills/
├── vms_app_backend -> /home/vms2/Vms/vms_app_backend/.agents/skills
├── vms_app_frontend -> /home/vms2/Vms/vms_app_frontend/.agents/skills
└── vms-engine -> /home/vms2/Vms/vms-engine/.agents/skills
```

Key project-specific skills:

| Sub-project | Notable skills |
|---|---|
| `vms_app_backend` | **Service generators (use these first for apps_v2 work):** |
| | `generate-apps-v2-vms-app-api-skill` — vms_app API endpoints, use cases, DTOs |
| | `generate-apps-v2-camera-control-skill` — camera control service (Hikvision, ONVIF, PTZ, audio) |
| | `generate-apps-v2-face-recognition-skill` — face recognition service |
| | `generate-apps-v2-worker-skill` — worker execution plane (manifest, recovery, reconcile) |
| | `generate-apps-v2-vms-app-monitoring-app-skill` — monitoring app integration |
| | **docs_v2 workflow (canonical docs → phase execution → implementation):** |
| | `seed-brainstorm-to-domain-docs-skill` — brainstorm → canonical product/domain docs |
| | `product-domain-to-phase-execution-skill` — product docs → phase execution bundle |
| | `implement-apps-v2-vms-app-phase-skill` — implement one phase, sync docs |
| | `maintain-docs-v2-skill` — create, backfill, reconcile docs_v2 bundles |
| | **Foundation:** `clean-ddd-hexagonal`, `fastapi-python`, `python-testing-patterns` |
| | **Infra:** `kafka-development`, `redis-development`, `postgresql-database-engineering` |
| | **⚠️ Always start with:** `using-superpowers` — must invoke before ANY response |
| `vms_app_frontend` | `apps-app-page-pattern`, `apps-mobile-screen-pattern`, `electron`, `expo-react-native-typescript`, `tanstack-router-best-practices`, `tanstack-query-best-practices` |
| `vms-engine` | `vms-engine-deepstream-feature-workflow`, `cmake`, `deepstream`, `docker` |

When working in a sub-project, prefer its own skills over generic ones. Read the sub-project's `AGENTS.md` for the full skill list.

### Agent workflow
- Root `reasonix.toml` defines the agent config, model, and loads skills from `.agents/skills/` in all three repos.
- Tool config folders (`.agents/`, `.codegraph/`, `.claude/`, `.gemini/`) are tracked in the workspace root.
- Use Reasonix plan mode for multi-step tasks: plan in the target repo, execute there, validate with that repo's commands.
- `.mcp.json` connects the `codegraph` MCP server for code navigation across all three repos.

### Code quality (cross-repo)
- **vms-engine**: `./scripts/format.sh --check` + `cmake --build build -- -j5` before PR.
- **vms_app_backend**: `make format && make typecheck && make test` from the app directory.
- **vms_app_frontend**: `pnpm lint && pnpm typecheck && pnpm test` from the app directory.

### Secrets
- Never commit secrets, credentials, or API keys. `.env` files are gitignored.

## Notes

- Root `README.md` gives a human-facing overview; `AGENTS.md` is the agent-facing index.
- Each sub-project's `AGENTS.md` is the source of truth for that repo's build commands, architecture rules, and coding conventions.
- Skill directories: `vms_app_backend/.agents/skills/` (55 skills), `vms_app_frontend/.agents/skills/` (57 skills), `vms-engine/.agents/skills/` (38 skills).
