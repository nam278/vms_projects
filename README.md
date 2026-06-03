# VMS — Video Management System

> GPU-accelerated video analytics platform: ingest, detect, track, record, and manage — from edge cameras to desktop.

## Overview

VMS is a three-tier video analytics system built on NVIDIA DeepStream:

- **vms-engine** — C++17 engine on DeepStream 8.0 that ingests RTSP camera streams, runs TensorRT AI inference and object tracking, and emits detection events via Redis/Kafka and RTSP outputs.
- **vms_app_backend** — Python FastAPI monorepo providing the control plane: camera topology management, event rules, face recognition, camera control, and worker lifecycle orchestration (DDD + Hexagonal Architecture).
- **vms_app_frontend** — TypeScript monorepo with an Electron desktop app for monitoring and operations, plus an Expo React Native mobile companion.

## Repository structure

```
Vms/
├── vms-engine/             ← C++ DeepStream analytics engine
├── vms_app_backend/        ← Python control-plane backend
├── vms_app_frontend/       ← TypeScript desktop & mobile frontend
├── .agents/                ← Agent skills (tracked)
├── .codegraph/             ← CodeGraph index (tracked)
├── .claude/                ← Claude config (tracked)
├── .gemini/                ← Gemini config (tracked)
├── reasonix.toml           ← Reasonix agent configuration
├── .mcp.json               ← MCP server config (codegraph)
├── AGENTS.md               ← Agent guidance (read this first)
├── README.md               ← This file
└── .gitignore
```

Each sub-directory is an **independent git repository** with its own build system, dependencies, and CI. Start in the relevant repo.

## Quick links

| Repo | README | AGENTS.md | Stack |
|------|--------|-----------|-------|
| `vms-engine/` | [README](vms-engine/README.md) | [AGENTS.md](vms-engine/AGENTS.md) | C++17 · CMake · DeepStream 8.0 · Docker |
| `vms_app_backend/` | [README](vms_app_backend/README.md) | [AGENTS.md](vms_app_backend/AGENTS.md) | Python 3.11 · FastAPI · uv · PostgreSQL |
| `vms_app_frontend/` | [README](vms_app_frontend/README.md) | [AGENTS.md](vms_app_frontend/AGENTS.md) | TypeScript · Electron · Expo · pnpm |

## Getting started

1. Clone each sub-repo into this workspace.
2. Read `AGENTS.md` for agent-assisted development guidance.
3. See each repo's own README for build and run instructions.

## Architecture

```
RTSP cameras → vms-engine → Redis/Kafka events → vms_app_backend → REST API → vms_app_frontend
```

- **vms-engine** processes video on GPU, publishes detection events.
- **vms_app_backend** stores topology, applies event rules, serves API.
- **vms_app_frontend** renders live views, dashboards, and config UIs.
