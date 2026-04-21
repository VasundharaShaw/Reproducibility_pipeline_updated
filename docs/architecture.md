# Architecture

## Pipeline Flow

run.sh
  -> pipeline/main.sh
        -> [Mode 1] prompt_for_input() -> process_repo()
        -> [Mode 2] process_sqlite_flow() -> process_repo() x N

process_repo()
  -> validate_repo()
  -> process_requirements()
  -> setup_pyenv_env()
  -> run_in_pyenv_env()
  -> compare_notebook_outputs()
  -> cleanup_pyenv_env()

## Directory Structure

| Path | Purpose |
|---|---|
| run.sh | Entry point - checks deps, launches pipeline |
| config/config.sh | All paths and settings |
| pipeline/main.sh | Main orchestrator |
| src/logging.sh | log(), now_sec(), elapsed_sec() |
| src/checks.sh | validate_repo() |
| src/db.sh | All SQLite operations + schema |
| src/pyenv.sh | Python version detection, venv setup/run/cleanup |
| src/requirements.sh | Merges requirements from files + notebook imports |
| src/notebooks.sh | Notebook output comparison |
| src/repo.sh | Per-repo orchestration + batch flow |
| analysis/ | Python comparison scripts and notebooks |
| data/input/ | Input repo lists |
| data/output/ | DB, logs, comparisons, cloned repos |

## Key Design Decisions

- pyenv + venv: each repo gets its own isolated environment at the correct Python version
- Non-fatal installs: packages install one-by-one, failures are logged but do not abort
- exec log format: SUCCESS/SUCCESS_WITH_ERRORS/EXEC_FAIL preserved for downstream compatibility
