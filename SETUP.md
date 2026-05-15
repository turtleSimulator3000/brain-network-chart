# Local Setup Guide

## Prerequisites

- **conda** (Miniconda/Anaconda) — [install here](https://docs.conda.io/en/latest/miniconda.html)
- **Node.js 18+** — [install here](https://nodejs.org)
- **Ollama** (only needed for agent/BIDS features) — [install here](https://ollama.com)


```
# Install Miniconda
curl -fsSL https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -o /tmp/miniconda.sh && bash /tmp/miniconda.sh -b -p "$HOME/miniconda3"

eval "$("$HOME/miniconda3/bin/conda" shell.bash hook)" && conda init bash && conda --version

# To activate conda in your current terminal without reopening it
source ~/.bashrc

# Install ollama
curl -fsSL https://ollama.com/install.sh | sh

# Confirm installation
ollama --version

# Pull the required model
ollama pull qwen3

# Comfirm ollama listening on port 11434
which ollama && ollama --version && curl -s http://localhost:11434/api/tags
```

---

## Step 1 — Install everything

```bash
cd brain-network-chart
bash install.sh
```

This creates the `brainchart` conda environment, installs all Python deps including `dcm2bids`/`dcm2niix`, and builds the frontend into `frontend/dist/`.

---

## Step 2 — Activate the environment

```bash
conda activate brainchart
```

---

## Step 3 — Start the backend

```bash
python mcp_server.py
```

The server starts on **http://localhost:8005** by default. Override the port with an env var if needed:

```bash
MCP_PORT=8010 python mcp_server.py
```

The backend also serves the pre-built frontend from `frontend/dist/`, so the full UI is available at the same URL.

---

## Step 4 (optional) — Frontend hot-reload dev server

If you're actively editing the UI, run Vite's dev server instead of relying on the static build:

```bash
cd frontend
npm run dev      # serves on http://localhost:5173
```

The Vite dev server proxies `/api` requests to the backend, so both must be running simultaneously.

---

## Step 5 (optional) — Enable LLM features

Pull a model into Ollama before starting the backend:

```bash
ollama pull qwen3          # recommended
ollama serve               # if not already running as a daemon
```

Without this, agent classification and validation endpoints exit with an error at startup. Set `DICOM2BIDS_MODEL=none` to run without Ollama in regex-only mode.

---

## Step 6 (optional) — BIDS local agent

If you need the BIDS conversion runner, start it in a separate terminal:

```bash
conda activate brainchart
python bids_local_agent.py   # listens on http://localhost:7789
```

The frontend detects this automatically and enables the "Run Locally" button.

---

## Quick reference

| What | URL |
|---|---|
| Full app (backend + built UI) | http://localhost:8005 |
| Frontend dev server | http://localhost:5173 |
| BIDS local agent | http://localhost:7789 |
