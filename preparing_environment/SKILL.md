---
name: preparing-environment
description: Set up the development environment for srt-slurm. Use when the user needs help with installation, dependencies, or environment configuration.
---

# Preparing Environment

When helping set up the srt-slurm development environment:

1. **Install dependencies**:
   ```bash
   uv sync
   ```

2. **Verify the setup**:
   ```bash
   make check   # Run lint + tests
   ```

3. **Key tools**:
   - `uv` for Python package management
   - `ruff` for linting and formatting
   - `pytest` for testing
   - `srtctl` CLI for job management

4. **SLURM environment**: If on a cluster, ensure SLURM environment variables are available (`SLURM_JOB_ID`, `SLURM_NODELIST`, etc.). For local development, tests mock these via `patch.dict(os.environ, ...)`.

5. **Configuration**: Cluster-level config lives in `srtslurm.yaml`. Check it exists and has correct cluster name and reporting endpoint settings.
