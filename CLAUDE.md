# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MiroFish is a Swarm Intelligence Engine for multi-agent AI prediction simulations. It ingests seed materials (reports, documents), builds knowledge graphs via GraphRAG, generates thousands of independent AI agent personas, runs parallel social media simulations (Twitter/Reddit) using CAMEL-AI OASIS, and produces prediction reports.

## Commands

```bash
# Setup
npm run setup:all              # Install all dependencies (npm + uv)
cp .env.example .env           # Then fill in API keys

# Development
npm run dev                    # Start backend + frontend concurrently
npm run backend                # Backend only (Flask on :5001)
npm run frontend               # Frontend only (Vite on :3000)
npm run build                  # Production frontend build

# Testing
cd backend && uv run pytest    # Run Python tests
```

## Architecture

**Full-stack app**: Python Flask backend + Vue 3 frontend, communicating via REST API.

### Backend (`backend/`)

Flask app using app factory pattern (`app/__init__.py`). Organized as:
- **`api/`** — Flask blueprints: `graph.py`, `simulation.py`, `report.py` (routes under `/api/graph`, `/api/simulation`, `/api/report`)
- **`services/`** — Core business logic. Largest files: `report_agent.py` (report generation with LLM tool use), `simulation_runner.py` (OASIS simulation orchestration), `oasis_profile_generator.py` (agent persona generation), `zep_tools.py` (graph querying)
- **`models/`** — `project.py` (project context), `task.py` (async task tracking). Both use singleton managers for in-memory state
- **`utils/`** — `llm_client.py` (OpenAI SDK wrapper supporting any compatible API), `file_parser.py`, `retry.py` (exponential backoff decorator)

**5-step workflow**: Upload files → Build knowledge graph → Generate agent profiles → Run OASIS simulation → Generate report. Each step has corresponding API endpoints and services.

**Key patterns**:
- Long-running operations (graph building, simulations, reports) use async task tracking with status polling
- LLM calls go through `utils/llm_client.py` which wraps the OpenAI SDK for any OpenAI-compatible API
- Zep Cloud is the memory graph database storing entities and relationships
- Simulation runner spawns child processes; needs cleanup on server shutdown

### Frontend (`frontend/`)

Vue 3 + Vite + Vue Router. No heavy state management (no Vuex/Pinia).
- **`views/`** — Page components: `MainView.vue` (5-step workflow), `SimulationView.vue`, `ReportView.vue`, `InteractionView.vue`
- **`components/`** — Step components (`Step1GraphBuild.vue` through `Step5Interaction.vue`), `GraphPanel.vue` (D3.js visualization)
- **`api/`** — Axios clients with centralized instance (`api/index.js`), modular clients per resource
- **`store/pendingUpload.js`** — Simple reactive state for file uploads

### External Dependencies

- **CAMEL-AI OASIS** (`camel-ai==0.2.78`, `camel-oasis==0.2.5`) — Multi-agent simulation framework
- **Zep Cloud** (`zep-cloud==3.13.0`) — Memory graph database for knowledge storage
- **LLM API** — Any OpenAI-compatible endpoint (default: Alibaba Qwen-plus via DashScope)

## Configuration

Environment variables in `.env` (see `.env.example`):
- `LLM_API_KEY`, `LLM_BASE_URL`, `LLM_MODEL_NAME` — Required LLM config
- `ZEP_API_KEY` — Required for knowledge graph storage
- `LLM_BOOST_*` — Optional secondary LLM for faster processing
- Flask config in `backend/app/config.py`: 50MB upload limit, allowed extensions `{pdf, md, txt, markdown}`, `JSON_AS_ASCII = False` for Chinese support

## Development Notes

- Backend uses **uv** as Python package manager (`pyproject.toml`, `uv.lock`)
- Python 3.11-3.12 required; Node.js 18+ required
- OASIS simulations consume many LLM tokens; test with <40 rounds initially
- Chinese language support is important: UTF-8 encoding handled throughout, especially on Windows
- The `<think>` tag stripping in content fields is necessary for reasoning models (MiniMax/GLM) — see commit `985f89f`
