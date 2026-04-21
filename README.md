---

## Usage

Run from `/home/jovyan`:

```bash
bash run.sh
```

`run.sh` checks all dependencies first, then launches the pipeline. You will be prompted to choose a mode:

### Mode 1 — Single repository

Enter a GitHub repository URL directly. You will be asked for:

- **GitHub repo URL** — e.g. `https://github.com/example/repo`
- **Notebook paths** — semicolon-separated paths to `.ipynb` files within the repo
- **Setup paths** *(optional)* — paths to `setup.py` files
- **Requirements paths** *(optional)* — paths to `requirements.txt` files

### Mode 2 — Batch mode

Processes repositories from `data/db.sqlite` (the source database containing 5241 repositories from Sheeba Samuel's original reproducibility study). Results are written back to the same database.

---

## How Environment Isolation Works

Each repository gets its own isolated Python environment via **pyenv + venv**:

1. Detects the required Python version by checking (in order):
   - `binder/runtime.txt`
   - `runtime.txt`
   - `.python-version`
   - `setup.py` / `setup.cfg` `python_requires` field
   - Falls back to Python 3.10
2. Installs that Python version via pyenv if not already present
3. Creates a fresh virtual environment under `~/.repo_venvs/<repo-name>/`
4. Installs the repo's dependencies into the venv
5. Executes all notebooks inside the venv
6. Cleans up the venv after execution

---

## Configuration

Edit `config/config.sh` or set environment variables before running:

```bash
export TARGET_COUNT=20    # Override batch size (default: 10)
bash run.sh
```

See `.env.example` for all available options.

---

## Database

`data/db.sqlite` is the source database from Sheeba Samuel's original study and contains 5241 repositories. **Do not overwrite this file.** Pipeline results (run status, notebook execution outcomes, reproducibility scores) are written to new tables within the same database.

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

## Related Publications

- Samuel, S., & Mietchen, D. (2024). Computational reproducibility of Jupyter notebooks from biomedical publications. *GigaScience*, 13, giad113. [DOI: 10.1093/gigascience/giad113](https://doi.org/10.1093/gigascience/giad113)
- Samuel, S., & Mietchen, D. (2024). FAIR Jupyter: A Knowledge Graph Approach to Semantic Sharing and Granular Exploration of a Computational Notebook Reproducibility Dataset. *TGDK*, 2(2), 4:1–4:24. [DOI: 10.4230/TGDK.2.2.4](https://doi.org/10.4230/TGDK.2.2.4)
- Samuel, S., & Mietchen, D. (2023). Dataset of a study of computational reproducibility of Jupyter notebooks from biomedical publications. *Zenodo*. [DOI: 10.5281/zenodo.8226725](https://doi.org/10.5281/zenodo.8226725)

---

## Acknowledgments

Supported by the **Jupyter4NFDI** project (DFG 567156310), **find.software** (DFG 567156310), **MaRDI** (DFG 460135501), **SeDOA** (DFG 556323977), and **HYP*MOL** (DFG 514664767).

---

## Contact

- **GitHub Issues**: [open an issue](https://github.com/VasundharaShaw/CPRPMC_test/issues)
- **Email**: sheeba.samuel@informatik.tu-chemnitz.de
- **Research Group**: [Distributed and Self-organizing Systems, TU Chemnitz](https://vsr.informatik.tu-chemnitz.de/)
