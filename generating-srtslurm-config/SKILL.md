---
name: generating-srtslurm-config
description: Generate srtslurm YAML configuration files for different GPU clusters, models, and deployment topologies. Use when the user wants to create a new recipe or config for running inference jobs.
---

# Generating srtslurm Config

When generating a new srtslurm configuration YAML file:

1. **Understand the requirements**: Ask about GPU type (H100/B200/H200), precision (FP8/FP16), model, sequence lengths, and topology (prefill/decode split).

2. **Reference existing recipes**: Read existing configs under `recipes/` to match the naming convention and structure. Use `Glob` to find similar configs:
   - `recipes/<gpu>-<precision>/<context_length>/<benchmark_type>/<config>.yaml`

3. **Follow the schema**: Configs must conform to the dataclass definitions in `src/srtctl/core/schema.py`. Key sections:
   - `name`: Job name
   - `backend`: SGLang or TRTLLM backend config
   - `resources`: GPU allocation (prefill/decode nodes, workers, GPUs per worker)
   - `infra`: Infrastructure placement (etcd/nats)
   - `benchmark`: Benchmark configuration

4. **Validate**: After generating, run `srtctl dry-run -f <config>.yaml` to verify the config produces valid commands.

5. **Naming convention**: Follow the pattern `<optimization-goal>-dep<decode_tp>-<topology>.yaml` (e.g., `max-tpt-dep8-1p1d.yaml`).
