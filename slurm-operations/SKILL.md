---
name: slurm-operations
description: Help with SLURM job operations including submitting, monitoring, canceling jobs, and debugging failures. Use when the user asks about SLURM job management.
---

# SLURM Operations

When helping with SLURM job operations:

1. **Submit jobs**:
   ```bash
   srtctl submit -f <config>.yaml
   ```

2. **Dry run** (preview generated commands without submitting):
   ```bash
   srtctl dry-run -f <config>.yaml
   ```

3. **Monitor jobs**:
   - `squeue -u $USER` to list running jobs
   - `sacct -j <job_id>` for job accounting info
   - Check logs in the job's log directory

4. **Debug failures**:
   - Check SLURM output/error files
   - Verify node allocation matches expected topology
   - Check health endpoints for worker readiness
   - Review `RuntimeContext` computed paths for log locations

5. **Common issues**:
   - GPU allocation mismatch: verify `gpus_per_node` matches cluster hardware
   - Node connectivity: check head node IP and etcd/nats placement
   - Container mount failures: verify paths in `container_mounts`
