# Project Overview — Brain Network Chart

A quick-start guide for engineers joining this project. Skim the headings first, then read the sections relevant to your work.

---

## 1. Project Summary

**Brain Network Chart** is an MCP (Model Context Protocol) server and web application for neuroimaging signal analysis and brain network research. It exposes a suite of computational neuroscience tools — graph hub detection, cross-frequency coupling (CFC), lifespan normative trajectory analysis, and clinical-grade statistical tests — through both a REST API and a React-based web UI.

Key capabilities:
- **DICOM → BIDS conversion** with LLM-powered series classification (using local Ollama models).
- **Cross-frequency coupling analysis** using harmonic wavelet decomposition on brain adjacency matrices.
- **Hub detection** in single and multi-network graphs via Laplacian eigenvector embedding.
- **Normative developmental curve fitting** against reference datasets (lifespan brain charts).
- **Statistical testing** — Pearson correlation, t-test / Mann-Whitney U, Benjamini-Hochberg FDR correction, and IQR outlier detection.
- **Literature validation** of analysis results against PubMed and OpenAlex.
- **MCP-compatible tool exposure**, enabling LLM agents (e.g., Claude via MCP) to call analysis functions directly.

---

## 2. Tech Stack

### Backend
| Layer | Technology |
|---|---|
| Language | Python 3.11+ |
| HTTP Framework | FastAPI + Uvicorn |
| MCP Protocol | `mcp >= 1.0.0` |
| Signal Processing | NumPy, SciPy, h5py, statsmodels |
| Neuroimaging | pydicom, dcm2bids, dcm2niix |
| LLM Integration | Ollama (local inference), pydantic-ai |
| HTTP Clients | httpx, requests |
| Data Validation | Pydantic v2 |
| Package Management | uv (preferred) or pip + conda |

### Frontend
| Layer | Technology |
|---|---|
| Framework | React 18, TypeScript 5.3 |
| Build Tool | Vite 5 |
| Charts | Recharts 2 |
| Dev Server | Vite dev server (hot reload) |

### Infrastructure
| Component | Technology |
|---|---|
| Container Runtime | Docker + Docker Compose |
| Static Serving | Nginx (production) |
| BIDS Processing | Dedicated container with FSL / Freesurfer |

---

## 3. Repository Structure

```
brain-network-chart/
│
├── mcp_server.py          # Main FastAPI app — all REST endpoints, rate limiting
├── mcp_server_adk.py      # ADK wrapper — re-exports tools via MCP protocol
│
├── tools.py               # Core analysis orchestration (CFC, hub detection, growth curves)
├── stats_tools.py         # Statistical tests (StatsToolkit class)
├── wavelets.py            # Harmonic wavelet decomposition, CFC computation
├── hub_detection.py       # Graph hub detection (Laplacian eigenvector method)
├── utils.py               # Shared utilities (vectorized correlation matrix)
│
├── agent_client.py        # Ollama LLM agent — tool selection and chaining
├── dicom2bids_agent.py    # LLM-powered DICOM series classifier + dcm2bids runner
├── bids_local_agent.py    # Local HTTP server (:7789) for BIDS job streaming (SSE)
├── validator.py           # ValidationAgent — literature-backed result validation
├── mcp_client.py          # MCP protocol client utilities
│
├── schema.json            # JSON Schema for tool parameter validation
├── install.sh             # One-shot environment setup script
├── pyproject.toml         # Python project metadata, uv/pip deps, pytest config
│
├── frontend/              # React + TypeScript web UI
│   ├── src/
│   │   ├── App.tsx        # Root component, state management
│   │   ├── api.ts         # All fetch calls to the backend
│   │   ├── types.ts       # TypeScript interfaces for all result types
│   │   ├── components/
│   │   │   ├── FileManager.tsx      # Upload / browse uploaded files
│   │   │   ├── StatsFormPanel.tsx   # Analysis forms (one per tool)
│   │   │   └── results/             # Per-result-type card components (Recharts)
│   └── ...
│
├── docker/
│   ├── docker-compose.yml      # Three-service stack (backend, frontend, bids-runner)
│   ├── Dockerfile.backend
│   ├── Dockerfile.frontend
│   └── nginx.conf
│
└── uploaded_files/        # Runtime file storage for user-uploaded CSVs / NPY files
```

---

## 4. Architecture Overview

```
┌──────────────────────────────────────────┐
│            React SPA (:8080)             │
│  FileManager │ StatsFormPanel │ Results  │
└────────────────────┬─────────────────────┘
                     │ fetch / SSE
┌────────────────────▼─────────────────────┐
│         mcp_server.py (:8004)            │
│  FastAPI REST API + rate limiting        │
│  ├── /upload, /list_files, /delete_file  │
│  ├── /run_correlation, /compare_groups   │
│  ├── /run_cfc_wavelet, /hub_detection    │
│  ├── /normative_analysis                 │
│  ├── /run_bids_conversion (SSE stream)   │
│  └── /validate, /pubmed_search, ...      │
│                                          │
│  delegates to:                           │
│  tools.py  stats_tools.py  hub_detection │
│  wavelets.py  utils.py                   │
└───────────┬──────────────────────────────┘
            │ spawns / calls
   ┌─────────▼──────────┐    ┌──────────────────────┐
   │  agent_client.py   │    │  bids_local_agent.py  │
   │  (Ollama LLM)      │    │  (:7789, SSE stream)  │
   └─────────┬──────────┘    └──────────┬────────────┘
             │                          │ Docker exec
   ┌─────────▼──────────┐    ┌──────────▼────────────┐
   │  validator.py      │    │  dicom2bids_agent.py   │
   │  PubMed / OpenAlex │    │  dcm2bids / dcm2niix  │
   └────────────────────┘    └───────────────────────┘

MCP Clients (e.g., Claude Desktop)
   └──► mcp_server_adk.py  ──► same analysis functions
```

**Key interaction patterns:**
- The frontend communicates exclusively through `api.ts` → `mcp_server.py`. No direct inter-module calls from the UI.
- Long-running jobs (BIDS conversion, CFC analysis) use **Server-Sent Events (SSE)** for streaming progress back to the browser.
- The `bids_local_agent.py` runs as a separate local process (`:7789`). The frontend detects its availability to enable the "Run Locally" button.
- `mcp_server_adk.py` wraps the same analysis functions to make them callable by MCP-compatible LLM clients without going through HTTP.
- `agent_client.py` uses Ollama locally; no external LLM API calls are made — all inference is on-device.

---

## 5. How to Run Locally

### Option A — Conda + direct Python (recommended for development)

```bash
# 1. Run the automated installer (creates the `brainchart` conda env)
bash install.sh

# 2. Activate the environment
conda activate brainchart

# 3. Start the backend (port 8004)
uvicorn mcp_server:http_app --host 0.0.0.0 --port 8004 --reload

# 4. In a separate terminal, start the frontend dev server
cd frontend
npm install
npm run dev          # Vite serves on http://localhost:5173
```

### Option B — Docker Compose (closest to production)

```bash
# Optionally set data paths
export DATA_INPUT_PATH=./input
export DATA_OUTPUT_PATH=./output

docker compose -f docker/docker-compose.yml up --build
# Frontend: http://localhost:8080
# Backend: http://localhost:8004 (internal, proxied via nginx)
# BIDS agent: http://localhost:7789
```

### Option C — uv (fast Python alternative)

```bash
uv sync
uvicorn mcp_server:http_app --host 0.0.0.0 --port 8004
```

> **Ollama requirement**: The LLM agent features (`agent_client.py`, `validator.py`, `dicom2bids_agent.py`) require a running Ollama instance with a compatible model pulled (e.g., `ollama pull medgemma` or `ollama pull qwen2.5`). Pure analysis endpoints work without Ollama.

> **Freesurfer license**: The BIDS runner container requires a valid `license.txt` mounted at `/fs_license/license.txt`. Without it, dcm2bids/FSL steps will fail. See [FILE_UPLOAD_GUIDE.md](FILE_UPLOAD_GUIDE.md) for data preparation.

---

## 6. Testing Strategy

The project uses **pytest** with the configuration in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
```

Run tests with:
```bash
conda activate brainchart
pytest
```

**Offline / mock mode**: `validator.py` includes a `MockOllamaLLM` fallback that activates automatically when Ollama is unreachable. This allows testing validation logic without a live LLM. Use this pattern when writing tests for agent-dependent code.

**Recommended test coverage areas** for new contributors:
- Unit tests for `stats_tools.py` functions (deterministic, no external deps).
- Unit tests for `wavelets.py` and `hub_detection.py` with synthetic adjacency matrices.
- Integration tests for `mcp_server.py` endpoints using `httpx.AsyncClient` and FastAPI's test client.

---

## 7. Development Workflow

### Adding a new analysis tool

1. **Implement the computation** in the appropriate module (`tools.py`, `stats_tools.py`, or a new module) and return a structured Pydantic model or dict.
2. **Register a FastAPI endpoint** in `mcp_server.py` following the existing request/response pattern (Pydantic input model → validated computation → JSON response).
3. **Expose via MCP** by adding a tool definition in `mcp_server_adk.py` if LLM clients should be able to call it.
4. **Add the TypeScript type** for the result in `frontend/src/types.ts` and extend the `ResultItem` discriminated union.
5. **Add the API call** in `frontend/src/api.ts`.
6. **Create a result card** in `frontend/src/components/results/` following the existing card pattern.
7. **Wire the form** in `StatsFormPanel.tsx` and add the result to `App.tsx` state handling.

### General branch / PR flow
- Work in feature branches off `main`.
- Keep backend and frontend changes in the same commit/PR if they are coupled.
- Verify the backend starts without errors (`uvicorn ... --reload`) and the frontend builds cleanly (`npm run build`) before opening a PR.

---

## 8. Common Pitfalls / Gotchas

| Pitfall | Detail |
|---|---|
| **Port conflicts** | Backend defaults to `:8004`, BIDS agent to `:7789`, frontend dev server to `:5173`. Docker Compose maps frontend to `:8080`. Check for conflicts if something appears unreachable. |
| **Ollama not running** | Agent and validation endpoints silently degrade or return errors when Ollama is not reachable. Start Ollama with `ollama serve` before testing agent features. |
| **Freesurfer license missing** | The `bids-runner` Docker container will fail at runtime without a valid `license.txt`. This is a hard requirement, not a warning. |
| **uploaded_files/ is runtime state** | Files uploaded through the UI land in `uploaded_files/`. This directory is not version-controlled. Analysis endpoints reference files by name from this directory — ensure the file exists before running an analysis. |
| **SSE streaming and proxies** | BIDS conversion and some analysis jobs use SSE. If you add a reverse proxy layer locally (e.g., ngrok), ensure it does not buffer responses — SSE requires flushing. |
| **`DATA_INPUT_PATH` / `DATA_OUTPUT_PATH` in Docker** | These env vars are optional but will default to `./input` and `./output` relative to the repo root. Create these directories before running `docker compose up` or mount them explicitly. |
| **Python 3.11 required** | Some dependencies (notably pydantic-ai and certain MCP internals) require Python 3.11+. Earlier versions will produce cryptic import errors. |
| **uv vs. pip** | `pyproject.toml` is configured for `uv`. Running plain `pip install -e .` may work but is not the tested path. Prefer `uv sync` or `bash install.sh`. |

---

## 9. Where to Find More Info

| Topic | Source |
|---|---|
| File upload formats and naming conventions | [FILE_UPLOAD_GUIDE.md](FILE_UPLOAD_GUIDE.md) |
| Validator agent design and prompting strategy | [Validator_Agent.md](Validator_Agent.md) |
| Docker deployment details | [docker/README.md](docker/README.md) |
| DICOM-to-BIDS conversion skill | [.github/skills/dicom2bids/SKILL.md](.github/skills/dicom2bids/SKILL.md) |
| MCP protocol specification | https://modelcontextprotocol.io |
| dcm2bids documentation | https://unfmontreal.github.io/Dcm2Bids |
| BIDS specification | https://bids-specification.readthedocs.io |
| Ollama model library | https://ollama.com/library |
| Example data files | [uploaded_files/](uploaded_files/) |
