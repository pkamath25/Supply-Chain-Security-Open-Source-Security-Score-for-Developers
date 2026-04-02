# 🛡️ DeepRisk OSS — Supply Chain Security Score for Developers

> **Real-time AI-powered security scoring of npm packages at install time — powered by Graph Attention Networks + LSTM.**

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python)](https://www.python.org/)
[![Node.js](https://img.shields.io/badge/Node.js-18%2B-green?logo=node.js)](https://nodejs.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100%2B-teal?logo=fastapi)](https://fastapi.tiangolo.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 📌 The Problem

Developers install third-party npm packages every day — often without knowing if they are malicious, abandoned, or have known vulnerabilities. Security risks are detected **too late** in the development lifecycle, after the package is already in the codebase.

## ✅ The Solution

**DeepRisk OSS** is a **CLI tool** that intercepts `npm install` and scores every package for supply-chain risk **before** it lands in your project. It uses a trained Graph Attention Network (GAT) + LSTM model that understands the entire npm dependency graph — not just isolated package metadata.

- 🔍 **Real-time risk scoring** at install / import time
- ⚡ **< 5ms** response for cached packages, **< 500ms** cold start
- 🤖 **AI model** trained on 40+ features per package (stars, commits, CVEs, dependency graph centrality, and more)
- 🛑 **Blocks HIGH risk installs** and suggests safer alternatives
- 📦 **Offline caching** — works without internet after first lookup
- 🔗 **OSV.dev integration** — live CVE data for packages not in the graph

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLI (Node.js)                        │
│   deeprisk install <pkg>  /  deeprisk scan <pkg1> <pkg2>    │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP (localhost:8000)
┌────────────────────────▼────────────────────────────────────┐
│                   Backend (FastAPI + Python)                 │
│                                                             │
│   /predict  →  L1 TTLCache → GAT+LSTM Model Inference       │
│   /predict/batch  →  Single graph pass → N O(1) lookups     │
│   /predict/{name} →  GET shortcut (curl / browser)          │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │     GAT + LSTM Model  (PyTorch Geometric)           │   │
│   │     • 40+ features per node                         │   │
│   │     • Dependency graph edges (500K+ packages)       │   │
│   │     • Temporal commit activity (LSTM)               │   │
│   │     • Val AUC: 0.992 | Train AUC: 0.999             │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   OSV.dev Fallback → Live CVE lookup for unknown packages   │
└─────────────────────────────────────────────────────────────┘
```

---

## 📁 Project Structure

```
deeprisk-oss/
├── backend/
│   ├── main.py                  # FastAPI app — all API routes
│   ├── config/
│   │   └── settings.py          # All config via env vars / .env
│   ├── model/
│   │   ├── architecture.py      # GAT + LSTM model definition
│   │   └── inference.py         # RiskPredictor — scoring engine
│   ├── services/
│   │   ├── cache.py             # Redis + in-process TTL cache
│   │   └── osv_service.py       # OSV.dev CVE integration
│   ├── schemas/                 # Pydantic request/response models
│   ├── utils/
│   │   ├── logger.py
│   │   └── rate_limiter.py      # Sliding window rate limiter
│   ├── data/                    # Pre-trained model + graph data
│   │   ├── best_model.pt        # Trained GAT+LSTM weights
│   │   ├── node_features.npy    # 40+ features per npm package
│   │   ├── edges.npy            # Dependency graph edges
│   │   └── ...
│   └── requirements.txt
└── cli/
    └── index.js                 # Node.js CLI — deeprisk command
```

---

## 🚀 Quick Start

### Prerequisites

| Tool | Version |
|------|---------|
| Python | 3.10 or higher |
| Node.js | 18 or higher |
| pip | latest |

---

### Step 1 — Start the Backend (Python API)

```bash
# 1. Clone the repo
git clone https://github.com/shubhamHULAGABALI/Supply-Chain-Security-Open-Source-Security-Score-for-Developers.git
cd Supply-Chain-Security-Open-Source-Security-Score-for-Developers

# 2. Create a virtual environment (recommended)
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. Install Python dependencies
pip install -r backend/requirements.txt

# 4. Start the API server
python backend/main.py
```

The server starts on **http://localhost:8000**. On first boot it loads the graph model (~250ms warm-up), then every prediction is under 5ms.

> **Tip:** You can also run with uvicorn directly:
> ```bash
> uvicorn backend.main:app --host 0.0.0.0 --port 8000
> ```

---

### Step 2 — Install the CLI (Node.js)

Open a **new terminal** and run:

```bash
cd cli
npm install
npm link          # makes "deeprisk" available globally
```

Verify it works:

```bash
deeprisk --help
deeprisk health
```

---

## 🖥️ CLI Usage

### Scan + Install (recommended)

Scans the package for risk **before** installing. Blocks HIGH risk packages automatically.

```bash
deeprisk install <package-name> [npm flags]
```

**Examples:**

```bash
deeprisk install express          
deeprisk install colors          
deeprisk install lodash --save-dev # passes npm flags through
```

---

### Scan Only (no install)

Check packages without installing anything.

```bash
deeprisk scan <pkg1> [pkg2 pkg3 ...]
```

```bash
deeprisk scan lodash express axios react
```

---

### Other Commands

| Command | Description |
|---------|-------------|
| `deeprisk install <pkg>` | Scan then install — blocks HIGH risk |
| `deeprisk scan <pkg...>` | Scan only, exit code 2 on HIGH risk |
| `deeprisk health` | Check if the backend API is running |
| `deeprisk config set <key> <val>` | Set API URL, timeout, etc. |
| `deeprisk cache clear` | Clear local offline cache |
| `deeprisk --help` | Show all commands |

---

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Safe — all packages passed |
| `1` | Install aborted (HIGH risk blocked) |
| `2` | HIGH risk found (scan only mode) |

---

### Environment Variables

Override CLI behaviour without editing config:

```bash
# Point CLI to a remote backend
DEEPRISK_API_URL=http://myserver:8000 deeprisk scan lodash

# Never block installs — warn only
DEEPRISK_NO_BLOCK=1 deeprisk install colors

# Adjust request timeout (ms)
DEEPRISK_TIMEOUT=3000 deeprisk scan axios
```

---

## 🔌 REST API Reference

The backend exposes a full REST API. You can use it directly from any tool, CI pipeline, or IDE plugin.

### Health Check

```bash
curl http://localhost:8000/health
```

```json
{ "status": "ok", "model_ready": true, "startup_ms": 312.4 }
```

---

### Score a Single Package

```bash
curl http://localhost:8000/predict/lodash
```

```bash
# Or via POST
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"package": "lodash", "with_neighbors": true}'
```

**Response:**
```json
{
  "package": "lodash",
  "risk_score": 87.3,
  "risk_prob": 0.873,
  "risk_label": "HIGH",
  "risk_tier": "Critical",
  "in_dataset": true,
  "val_auc": 0.992,
  "inference_ms": 3.1,
  "top_neighbors": [...],
  "osv": { "vuln_count": 4, "cves": [...] },
  "explanation": "High dependency centrality + known CVEs",
  "alternatives": ["ramda", "underscore"],
  "cached": false
}
```

---

### Batch Scan (up to 50 packages)

```bash
curl -X POST http://localhost:8000/predict/batch \
  -H "Content-Type: application/json" \
  -d '{"packages": ["lodash", "express", "chalk", "axios"]}'
```

```json
{
  "results": [...],
  "total_ms": 18.4,
  "high_risk_count": 2,
  "packages_in_dataset": 4
}
```

---

### Interactive API Docs

Open your browser at **http://localhost:8000/docs** for the full Swagger UI — try every endpoint live.

---

## ⚙️ Configuration (Backend)

All backend settings can be set via environment variables or a `.env` file in the project root:

```env
# .env (create this file in the project root)

# Model & data
DEEPRISK_DATA_DIR=backend/data
DEEPRISK_DEVICE=cpu               # or "cuda" for GPU

# Server
HOST=0.0.0.0
PORT=8000

# Cache (Redis optional — falls back to in-memory if not set)
REDIS_URL=redis://localhost:6379/0
CACHE_TTL_SECONDS=86400           # 24 hours

# Rate limiting
RATE_LIMIT_PER_MINUTE=120

# OSV.dev live CVE lookup
OSV_FALLBACK=1
OSV_TIMEOUT_S=2.0

# Auth (optional — leave unset for open access)
DEEPRISK_API_KEY=your-secret-key

# Logging
LOG_LEVEL=INFO
```

---

## 🧠 Model Details

| Property | Value |
|----------|-------|
| Architecture | GAT (Graph Attention Network) + LSTM |
| Framework | PyTorch + PyTorch Geometric |
| Features per package | 40+ (metadata, git activity, CVEs, graph centrality) |
| Training AUC | **0.999** |
| Validation AUC | **0.992** |
| Validation F1 | **0.819** |
| Optimal threshold | 0.768 |
| Inference (warm) | < 5ms |
| Inference (cold) | < 500ms |

**Key features used for scoring:**

- Package metadata: maintainer count, license, description length, age
- GitHub signals: stars, forks, open issues, commit frequency (7/30/90/180/365 days)
- Temporal trends: commit slope over 104 weeks, recent vs older activity ratio
- Vulnerability data: CVE count from OSV.dev
- Graph signals: in-degree centrality, PageRank in the dependency graph

---

## 🔒 Constraints & Guardrails

This tool is designed to **never block your workflow**:

- ✅ **Low latency** — all predictions served from cache in < 5ms
- ✅ **Offline support** — CLI caches results locally for 7 days
- ✅ **Non-blocking mode** — set `DEEPRISK_NO_BLOCK=1` to warn without blocking
- ✅ **Unknown packages** — falls back to OSV.dev live CVE lookup gracefully
- ✅ **Redis optional** — automatically uses in-memory cache if Redis is unavailable

---

##  💡 Problem & Solution Overview

This project was built for the **Supply Chain Security: Open-Source Security Score for Developers** track.

**Problem statement:** Developers rely on third-party libraries that may be malicious or unmaintained, with risks detected too late in the lifecycle.

**What we built:** A CLI tool + IDE-ready API that provides real-time security scoring of packages during install or import, suggesting safer alternatives — without blocking builds.

---

## 🤝 Contributing

Pull requests are welcome! For major changes, please open an issue first.

1. Fork the repo
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -m "Add my feature"`
4. Push and open a PR

---

## 📄 License

MIT © [Shubham Hulagabali](https://github.com/shubhamHULAGABALI)
