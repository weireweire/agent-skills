---
name: using-srtslurm
description: Help with srtslurm job operations including submitting, monitoring, canceling jobs, and debugging failures. Use when the user asks about SLURM job management.
---

# SLURM Operations

When helping with SLURM job operations:

1. **Submit jobs**:
   ```bash
   srtctl apply -f <config>.yaml -o OUTPUT_DIR
   ```

   **Job naming convention**: When submitting jobs, use a descriptive name with timestamp format: `MMDD-HHMM-<description>` by `-o`.

   **Exclude bad nodes**: Add `sbatch_directives` to the config yaml to exclude known-bad nodes:
   ```yaml
   sbatch_directives:
     exclude: "gpu-[5,8]"
   ```

2. **Dry run** (preview generated commands without submitting):
   ```bash
   srtctl dry-run -f <config>.yaml
   ```


4. **Debug failures**:
   - Check srt-slurm/outputs/ for log. Error can occur in sweep_xxxx.log or prefill/decode/agg log file in logs dir.

5. **Common issues**:
   - GPU allocation mismatch: verify `gpus_per_node` matches cluster hardware
   - Node connectivity: check head node IP and etcd/nats placement
   - Container mount failures: verify paths in `container_mounts`

can use --setup-script to do custom operation in container. Can use extra_mount to add custom mount to container. combine this two can install custom package.

when debugging, can use load-format: dummy and disable-cuda-graph to speed up if not relavent.
