# Containing the Reproducibility Gap

**Automated Repository-Level Reproducibility Assessment for Scholarly Jupyter Notebooks**

[![License: GPL-3.0](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![Platform: NFDI JupyterHub](https://img.shields.io/badge/platform-NFDI%20JupyterHub-orange.svg)](https://hub.nfdi-jupyter.de)

---

## Overview

This pipeline automatically clones GitHub repositories containing Jupyter notebooks, re-executes them in isolated Python environments, and measures how reproducible the results are. It is designed to run on the **[NFDI JupyterHub](https://hub.nfdi-jupyter.de)** via repo2docker — no local Docker installation needed.

Results are stored in a SQLite database for downstream analysis.

### What it does

1. **Clones** a GitHub repository containing Jupyter notebooks
2. **Detects** the required Python version from the repo's metadata
3. **Creates** an isolated pyenv + venv environment per repository
4. **Executes** each notebook via `nbconvert`
5. **Compares** original vs. re-executed outputs using `nbdime`
6. **Stores** cell-level reproducibility scores in a SQLite database

---

## Running on NFDI JupyterHub (recommended)

This is the primary way to run the pipeline.

1. Go to [hub.nfdi-jupyter.de](https://hub.nfdi-jupyter.de/hub/home)
2. Click **Start Server** and choose **Repo2docker (Binder)**
3. Fill in the form:
   - **Repository URL**: `https://github.com/VasundharaShaw/CPRPMC-pyenv-Vasu`
   - **Git ref**: `main`
   - **Flavor**: `4GB RAM, 1 vCPU` (minimum recommended)
4. Click **Start** — the environment will build automatically
5. Once JupyterLab opens, launch a terminal and run:

```bash
cd scripts
bash main.sh
```

The script will prompt you to choose a mode (see [Usage](#usage) below).

---

## Repository Structure

```
CPRPMC-pyenv-Vasu/
├── binder/                  # repo2docker configuration (NFDI build)
│   ├── Dockerfile           # Ubuntu 22.04 + Python 3.10 + pyenv
│   ├── apt.txt              # System package dependencies
│   └── postBuild            # Post-build setup (pyenv init)
│
├── config/
│   └── config.sh            # Pipeline configuration (paths, directories)
│
├── data/
│   └── db/
│       └── db.sqlite        # SQLite results database
│
├── lib/                     # Pipeline library functions
│   ├── pyenv.sh             # Python version detection + venv isolation
│   ├── repo.sh              # Repository cloning and processing
│   ├── requirements.sh      # Dependency extraction
│   ├── notebooks.sh         # Notebook execution via nbconvert
│   ├── db.sh                # Database operations
│   ├── checks.sh            # Pre-flight validation checks
│   └── logging.sh           # Logging utilities
│
├── scripts/
│   ├── main.sh              # Pipeline entry point
│   ├── compare_notebook.py  # Output comparison (nbdime)
│   └── nbprocess/           # Notebook processing utilities
│
├── analysis/
│   └── analyse_reporesults.ipynb  # Explore results from db.sqlite
│
└── requirements.txt         # Python dependencies for the pipeline itself
```

---

## Usage

Run the pipeline from the `scripts/` directory:

```bash
cd scripts
bash main.sh
```

You will be prompted to choose a mode:

### Mode 1 — Single repository (custom URL)

Enter a GitHub repository URL directly and the pipeline will process it immediately. You will be asked for:

- **GitHub repo URL** — e.g. `https://github.com/example/repo`
- **Notebook paths** — semicolon-separated paths to `.ipynb` files within the repo
- **Setup paths** *(optional)* — paths to `setup.py` files
- **Requirements paths** *(optional)* — paths to `requirements.txt` files

### Mode 2 — Batch mode (SQLite flow)

Processes all repositories already registered in `data/db/db.sqlite`. This is the main mode for large-scale reproducibility studies.

---

## How Environment Isolation Works

Unlike the original pipeline (which used Docker-in-Docker), this version uses **pyenv + per-repository virtual environments**. This makes it compatible with platforms like NFDI JupyterHub where Docker is not available.

For each repository, the pipeline:

1. Detects the required Python version by checking (in order):
   - `binder/runtime.txt`
   - `runtime.txt`
   - `.python-version`
   - `setup.py` / `setup.cfg` `python_requires` field
   - Falls back to Python 3.10
2. Installs that Python version via pyenv (if not already installed)
3. Creates a fresh virtual environment under `~/.repo_venvs/<repo-name>/`
4. Installs the repo's dependencies into that venv
5. Executes all notebooks inside the venv
6. Cleans up the venv after execution

---

## Database Schema

Results are stored in `data/db/db.sqlite` with the following tables:

| Table | Description |
|---|---|
| `repositories` | Repository metadata (URL, notebook count, requirements) |
| `repository_runs` | Per-run status, timestamps, duration |
| `notebook_executions` | Per-notebook execution results and errors |
| `notebook_reproducibility_metrics` | Cell-level reproducibility scores |

### Example queries

```sql
-- Repositories with highest average reproducibility score
SELECT r.repository, AVG(nrm.reproducibility_score) AS avg_score
FROM repositories r
JOIN notebook_reproducibility_metrics nrm ON r.id = nrm.repository_id
GROUP BY r.id
ORDER BY avg_score DESC
LIMIT 10;

-- Most common failure types
SELECT error_type, COUNT(*) AS count
FROM notebook_executions
WHERE execution_status NOT IN ('SUCCESS', 'SUCCESS_WITH_ERRORS')
GROUP BY error_type
ORDER BY count DESC;

-- Success rate by whether repo had a requirements.txt
SELECT
    CASE WHEN requirements IS NOT NULL THEN 'With requirements.txt'
         ELSE 'Without requirements.txt' END AS req_status,
    COUNT(*) AS total,
    SUM(CASE WHEN run_status = 'SUCCESS' THEN 1 ELSE 0 END) AS successful
FROM repositories r
JOIN repository_runs rr ON r.id = rr.repository_id
GROUP BY req_status;
```

To explore results interactively, open `analysis/analyse_reporesults.ipynb` in JupyterLab.

---

## Configuration

Edit `config/config.sh` to change default paths:

```bash
DB_FILE="data/db/db.sqlite"     # SQLite database location
REPOS_DIR="data/repositories"   # Where repos are cloned
COMP_DIR="data/comparisons"     # Where notebook comparisons are stored
LOG_DIR="data/logs"             # Execution logs
```

---

## Related Publications

- Samuel, S., & Mietchen, D. (2024). Computational reproducibility of Jupyter notebooks from biomedical publications. *GigaScience*, 13, giad113. [DOI: 10.1093/gigascience/giad113](https://doi.org/10.1093/gigascience/giad113)
- Samuel, S., & Mietchen, D. (2024). FAIR Jupyter: A Knowledge Graph Approach to Semantic Sharing and Granular Exploration of a Computational Notebook Reproducibility Dataset. *TGDK*, 2(2), 4:1–4:24. [DOI: 10.4230/TGDK.2.2.4](https://doi.org/10.4230/TGDK.2.2.4)
- Samuel, S., & Mietchen, D. (2023). Dataset of a study of computational reproducibility of Jupyter notebooks from biomedical publications. *Zenodo*. [DOI: 10.5281/zenodo.8226725](https://doi.org/10.5281/zenodo.8226725)

---

## Acknowledgments

Supported by the **Jupyter4NFDI** project (DFG 567156310), **find.software** (DFG 567156310), **MaRDI** (DFG 460135501), **SeDOA** (DFG 556323977), and **HYP*MOL** (DFG 514664767).

---

## Contact

- **GitHub Issues**: [open an issue](https://github.com/VasundharaShaw/CPRPMC-pyenv-Vasu/issues)
- **Email**: sheeba.samuel@informatik.tu-chemnitz.de
- **Research Group**: [Distributed and Self-organizing Systems, TU Chemnitz](https://vsr.informatik.tu-chemnitz.de/)
