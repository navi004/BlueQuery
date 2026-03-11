# BlueQuery

Backend logic and code for the **BlueQuery SIH2026** project — an intelligent oceanographic data assistant powered by AI agents.

## 📋 Overview

BlueQuery is a sophisticated system designed to process oceanographic queries about ARGO float data using a multi-agent CrewAI framework. The backend provides:

- **Agentic Query Processing**: A three-tier agent system (Prompt Guard, Query Processor, Output Formatter)
- **Voice Input Support**: Record and transcribe audio queries using Gemini
- **Database Integration**: MCP-based SQLite database access for ARGO float datasets
- **FastAPI REST Endpoints**: Production-ready API for frontend integration
- **Data Visualization**: Support for chart generation and data summarization
- **Safety Guardrails**: Built-in prompt validation and unsafe input detection

---

## 🗂️ Data Source

**Argo Float Program — Global Ocean Observation Network**

The Argo program operates a global array of ~4,000 autonomous profiling floats that measure temperature, salinity, and pressure throughout the world's oceans. Data is freely available via the [Argo Data Management](https://argo.ucsd.edu/data/) portal.

> **Note**: Due to file size constraints, `incois_floats_combined.xlsx` contains a representative sample of the processed INCOIS subset. The full dataset was used during development and model training.

### Index Catalog Files

The Argo system provides global index catalog files that act as a data discovery layer — listing all available NetCDF profile files without requiring full dataset downloads.

| File | Description |
|------|-------------|
| `ar_index_global_meta.txt.gz` | Float metadata catalog |
| `ar_index_global_prof.txt.gz` | Profile measurement index |
| `ar_index_global_traj.txt.gz` | Float trajectory index |
| `ar_index_global_tech.txt.gz` | Technical parameter index |
| `argo_bio_profile_index.txt` | Biogeochemical profile index |
| `argo_synthetic_profile_index.txt` | Synthetic profile index |

Each row in these files describes a single NetCDF file with fields: `file`, `date`, `latitude`, `longitude`, `ocean`, `profiler_type`, `institution`, `date_update`.

## 📓 Notebooks

| Notebook | Description |
|----------|-------------|
| `DATA.ipynb` | Initial data pipeline — Argo index parsing, timestamp normalization, metadata feature engineering |
| `DATA_updated.ipynb` | Refined pipeline — INCOIS subsetting, NetCDF file acquisition, xarray flattening to relational tables |
| `SIH_DATA_RETRIEVAL.ipynb` | SIH-specific retrieval pipeline — DAC filtering, cycle limiting, SQLite database generation for deployment |

These notebooks document the full data engineering workflow from raw Argo catalogs to the structured SQLite database used by the AI agent backend.

---

## 🔧 Data Pipeline

The data pipeline transforms raw Argo scientific data into a structured SQLite database ready for AI-powered querying.

### Pipeline Overview

```
Argo Index Catalogs (.txt.gz)
        │
        ▼
1. Decompression & Parsing
        │
        ▼
2. Normalization & Feature Engineering
        │
        ▼
3. Subsetting Strategy (DAC + Cycle filter)
        │
        ▼
4. NetCDF File Acquisition
        │
        ▼
5. NetCDF Flattening → Relational Tables
        │
        ▼
SQLite Database (argo_db.db)
        │
        ▼
MCP Adapter → AI Agent → FastAPI → Frontend
```

---

### Stage 1 — Decompression & Parsing

Index files are provided as `.txt.gz` compressed archives. The pipeline decompresses them, skips metadata comment lines (prefixed with `#`), and parses them into DataFrames.

```python
import gzip
import pandas as pd

with gzip.open("ar_index_global_prof.txt.gz", "rt") as f:
    lines = [l for l in f if not l.startswith("#")]

df = pd.read_csv(pd.io.common.StringIO("".join(lines)))
```

Output: DataFrame with millions of rows, 8–12 columns per index file.

---

### Stage 2 — Normalization & Feature Engineering

**Timestamp normalization:**
```python
df["date"] = pd.to_datetime(df["date"], errors="coerce")
df["date_update"] = pd.to_datetime(df["date_update"], errors="coerce")
```

**Metadata extraction from file path:**

The `file` column encodes hidden structure not available as separate columns. For example:
```
incois/3901702/profiles/R3901702_015.nc
```

Extracted fields:

| Derived Feature | Extracted Value | Meaning |
|-----------------|-----------------|---------|
| `dac` | `incois` | Data Assembly Center |
| `float_id` | `3901702` | Unique float identifier |
| `profile_type` | `R` | Real-time (R) or Delayed (D) mode |
| `cycle_number` | `15` | Profile cycle number |

```python
import re

df["dac"] = df["file"].apply(lambda x: x.split("/")[0])
df["float_id"] = df["file"].apply(lambda x: x.split("/")[1])
df["profile_type"] = df["file"].apply(lambda x: x.split("/")[-1][0])
df["cycle_number"] = df["file"].apply(
    lambda x: int(re.search(r"_(\d+)\.nc$", x).group(1))
)
```

---

### Stage 3 — Subsetting Strategy

The global Argo dataset contains 4M+ profiles across TB-scale data. Two filters were applied to produce a manageable, scientifically coherent subset.

**Filter 1 — DAC Selection (INCOIS)**

INCOIS (Indian National Centre for Ocean Information Services) manages floats deployed primarily in the Indian Ocean, providing a regionally coherent dataset.

```python
df_subset = df[df["dac"] == "incois"]
```

**Filter 2 — Cycle Limitation**

Each float can produce 200–400 cycles. Cycles 1–50 were retained to capture the launch and early operational phases while keeping storage manageable.

```python
df_subset = df_subset[
    (df_subset["cycle_number"] >= 1) &
    (df_subset["cycle_number"] <= 50)
]
```

Result: DAC-specific index files — `meta_incois.csv`, `prof_incois.csv`, `traj_incois.csv`, `tech_incois.csv` — reducing millions of rows to thousands.

---

### Stage 4 — NetCDF File Acquisition

Using the filtered `file` column, download URLs are reconstructed and files are downloaded preserving the original directory hierarchy.

```python
BASE_URL = "https://data-argo.ifremer.fr/dac/"

for file_path in df_subset["file"]:
    url = BASE_URL + file_path
    local_path = f"subset_dac_downloads/{file_path}"
    os.makedirs(os.path.dirname(local_path), exist_ok=True)
    # download and save
```

Local structure preserved:
```
subset_dac_downloads/
└── incois/
    └── 3901702/
        └── profiles/
            ├── R3901702_001.nc
            ├── R3901702_002.nc
            └── ...
```

---

### Stage 5 — NetCDF Flattening

NetCDF files contain multi-dimensional arrays. For example, temperature is stored as:

```
TEMP(N_PROF, N_LEVELS)  →  e.g., TEMP[0, 0..119]  (120 depth levels)
```

This must be converted to relational rows for SQL storage.

```python
import xarray as xr

ds = xr.open_dataset("R3901702_015.nc")
df = ds.to_dataframe().reset_index()
```

**Before flattening (NetCDF):**
```
TEMP[profile=0, level=0] = 28.2
TEMP[profile=0, level=1] = 27.9
...
```

**After flattening (SQL-ready):**

| platform | cycle | pressure | temperature | salinity |
|----------|-------|----------|-------------|----------|
| 3901702  | 15    | 10.0     | 28.2        | 35.1     |
| 3901702  | 15    | 20.0     | 27.9        | 35.2     |
| 3901702  | 15    | 30.0     | 27.6        | 35.3     |

Each depth level becomes one row, preserving full scientific precision.

---

## 🗄️ Database Schema

The pipeline produces a structured SQLite database (`argo_db.db`) with four relational tables.

### `meta_rel` — Float Metadata
One row per float.

| Column | Description |
|--------|-------------|
| `PLATFORM_NUMBER` | Unique float identifier |
| `PLATFORM_TYPE` | Hardware type/model |
| `PLATFORM_FAMILY` | Float family classification |
| `PLATFORM_MAKER` | Manufacturer |
| `PROJECT_NAME` | Deployment project name |
| `PI_NAME` | Principal Investigator |
| `LAUNCH_DATE` | Deployment date |
| `LAUNCH_LATITUDE` | Deployment latitude |
| `LAUNCH_LONGITUDE` | Deployment longitude |

### `traj_rel` — Float Trajectory
One row per trajectory observation.

| Column | Description |
|--------|-------------|
| `PLATFORM_NUMBER` | Float identifier |
| `LATITUDE` | Observed latitude |
| `LONGITUDE` | Observed longitude |
| `JULD` | Julian date timestamp |
| `CYCLE_NUMBER` | Profile cycle |
| `POSITION_QC` | Position quality flag |
| `DATA_MODE` | Real-time or delayed mode |

### `prof_rel` — Profile Measurements
One row per depth level measurement.

| Column | Description |
|--------|-------------|
| `PLATFORM_NUMBER` | Float identifier |
| `CYCLE_NUMBER` | Profile cycle |
| `PRES` | Pressure (dbar) |
| `TEMP` | Temperature (°C) |
| `PSAL` | Salinity (PSU) |
| `PRES_QC` | Pressure quality flag |
| `TEMP_QC` | Temperature quality flag |
| `PSAL_QC` | Salinity quality flag |

### `tech_rel` — Technical Parameters
One row per technical metadata entry.

| Column | Description |
|--------|-------------|
| `PLATFORM_NUMBER` | Float identifier |
| `TECHNICAL_PARAMETER_NAME` | Parameter name |
| `TECHNICAL_PARAMETER_VALUE` | Parameter value |
| `CYCLE_NUMBER` | Associated cycle |

---

## 🏗️ Architecture

### Core Components

#### 1. **Agent-Based Pipeline** (`dev_notebooks/devsan.py`)
- **Prompt Guard Agent**: Validates user queries for safety and relevance
- **Query Processor Agent**: Interprets queries and fetches/analyzes ARGO data
- **Output Formatter Agent**: Formats responses as clean, Markdown-structured output

#### 2. **Voice Input System** (`dev_notebooks/devsan-2.py`)
- Records audio from microphone
- Transcribes using Google Gemini API
- Feeds transcript into the agent pipeline

#### 3. **MCP Integration** (`dev_notebooks/devsan-mcp.py`)
- Model Context Protocol (MCP) adapter for SQLite database
- Executes SQL queries against ARGO float database
- Supports schema-aware data retrieval

#### 4. **Visualization Module** (`dev_notebooks/devsan-viz.py`)
- Generates charts using `@antv/mcp-server-chart`
- Visualizes oceanographic data (salinity profiles, trajectories, etc.)

#### 5. **FastAPI Server** (`main.py`, `main2.py`, `main3.py`, `main4.py`, `main5.py`)
- REST API endpoint: `POST /query`
- Persistent MCP connection (main2.py)
- Request caching and rate limiting
- Configurable database paths and timeouts

---

## 📁 Project Structure

```
BlueQuery-backend/
├── dev_notebooks/
│   ├── devsan.py              # Base agent pipeline (no MCP)
│   ├── devsan-2.py            # Voice input + agent pipeline
│   ├── devsan-mcp.py          # MCP + SQLite integration
│   ├── devsan-mcp-rag.py      # RAG variant with MCP
│   ├── devsan-viz.py          # Visualization variant
│   └── local_devsan.py        # Local LLM variant (Ollama)
├── main.py                    # FastAPI endpoint (basic)
├── main2.py                   # FastAPI with persistent MCP
├── main3.py                   # FastAPI variant (no formatter)
├── main4.py                   # FastAPI with schema details
├── main5.py                   # FastAPI with logging suppression
├── data/
│   ├── sample_argo_dataset.xlsx      # Sample float data
│   ├── incois_floats_combined.xlsx   # Processed INCOIS subset (sample)
│   └── R1902671_072.nc               # Sample raw NetCDF file
├── tests/
│   ├── fapi-test.py           # FastAPI test
│   ├── gemini_voice_test.py   # Gemini transcription test
│   └── voiceInputTest.py      # Whisper transcription test
├── ARGO_DATA/                 # Raw index and NetCDF files
├── notebooks/
│   ├── DATA.ipynb                 # Initial data pipeline — index parsing, normalization, feature engineering
│   ├── DATA_updated.ipynb         # Refined pipeline — subsetting, NetCDF acquisition, flattening to relational tables
│   └── SIH_DATA_RETRIEVAL.ipynb  # SIH-specific data retrieval — DAC filtering, SQLite database generation
├── database/                  # SQLite database files
├── pyproject.toml             # Project dependencies
└── README.md
```

---

## 🚀 Quick Start

### Prerequisites

- Python 3.11+
- [Node.js](https://nodejs.org/) (for MCP server: `@executeautomation/database-server`)
- [Google Gemini API Key](https://aistudio.google.com/)
- SQLite database file with ARGO data

### Installation

```bash
git clone https://github.com/navi004/BlueQuery-backend.git
cd BlueQuery-backend
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Environment Variables

```bash
export GEMINI_API_KEY="your-api-key-here"
export ARGO_DB_PATH="/path/to/argo_database.db"
```

### Running the Application

```bash
# Interactive CLI
python dev_notebooks/devsan.py

# FastAPI Server
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# MCP-Enhanced Server
uvicorn main2:app --reload --host 0.0.0.0 --port 8000
```

### API Usage

**Request:**
```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "Show me salinity profiles near the equator in March 2023"}'
```

**Response:**
```json
{
  "query": "Show me salinity profiles near the equator in March 2023",
  "result": "## Salinity Profiles Analysis\n\n### Geographic Focus\n- **Region**: Equatorial Ocean\n- **Timeframe**: March 2023\n...",
  "cached": false
}
```

---

## 🔧 Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `GEMINI_API_KEY` | Required | Google Gemini API key |
| `ARGO_DB_PATH` | `./agro_db_5_floats.db` | Path to SQLite database |
| `MAX_CONCURRENT_REQUESTS` | `4` | Max simultaneous requests |
| `MCP_CONNECT_TIMEOUT` | `20` | MCP connection timeout (seconds) |
| `REQUEST_TIMEOUT_SECONDS` | `180` | Request processing timeout |
| `THREAD_POOL_WORKERS` | `4` | Thread pool size |

---

## 📦 Dependencies

```
crewai[tools]>=0.193.2
python-dotenv>=1.1.1
fastapi
uvicorn
xarray
netCDF4
google-genai
sounddevice
soundfile
whisper (optional, for local transcription)
```

---

## 🔐 Safety & Guardrails

- **Prompt Validation**: All queries pass through a safety filter
- **UNSAFE PROMPT Detection**: Blocks irrelevant or malicious queries
- **Structured Output**: Markdown formatting with clear sections
- **Rate Limiting**: Semaphore-based concurrency control

---

## 🧪 Testing

```bash
pytest tests/ -v
python tests/gemini_voice_test.py
python tests/fapi-test.py
```

---

## 🎓 Project Context

**BlueQuery** is part of the **Smart India Hackathon 2025/2026** initiative (Semi-finalist), focusing on making oceanographic data accessible through AI-powered conversational interfaces. The project addresses the challenge of querying complex scientific NetCDF data without domain expertise, enabling researchers, students, and policymakers to interact with Argo float data using plain language.

---

## 📝 License

This project is licensed under the **MIT License**. See LICENSE file for details.

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📧 Support

For questions or issues, please:
- Open a GitHub [Issue](https://github.com/YasirAhmadX/BlueQuery-backend/issues)
- Contact the maintainer: [@YasirAhmadX](https://github.com/YasirAhmadX)

## 🎓 Project Context

**BlueQuery** is part of the **Smart India Hackathon 2025/2026** initiative, focusing on oceanographic data analysis and accessibility through AI-powered conversational interfaces.

---

**Last Updated**: March 2026  
---

## 📌 Key Highlights of This README

1. **Clear Overview**: Explains what the project does
2. **Architecture Breakdown**: Details each component's role
3. **File Structure**: Easy to understand the codebase
4. **Database Schema**: Explains the ARGO data structure
5. **Quick Start Guide**: Step-by-step setup instructions
6. **API Examples**: Curl requests showing how to use it
7. **Configuration**: Environment variables and LLM options
8. **Safety Features**: Highlights guardrails
9. **Testing**: Instructions for running tests
10. **Context**: Mentions the SIH2026 hackathon




