---
name: gaussian-16
description: Submit and monitor Gaussian 16 jobs via the unified HPC API using software=GAUSSIAN and a custom cmd override that runs g16 plus formchk. Use after uploading a `.gjf` or input ZIP to a reachable HTTP URL.
---

# Gaussian 16 Unified HPC Skill

## Workflow
1. **Prepare the input** as a `.gjf` or ZIP bundle reachable by HTTP (`params.inputfile`).
2. **Configure this skill** by copying `scripts/config/gaussian_config.example.json` to `scripts/config/gaussian_config.json` and filling in production values.
3. **Submit a Gaussian job** via the helper script:
   ```bash
   cd skills/gaussian-16/scripts
   ./gaussian_client.py --config config/gaussian_config.json \
     submit http://example.com/path/input.gjf --job-type opt
   # -> prints job_id + task_id
   ```
4. **Poll job status** with the returned integer `job_id`:
   ```bash
   ./gaussian_client.py --config config/gaussian_config.json poll <job_id>
   ```
5. **Download the result ZIP** with the returned `task_id`:
   ```bash
   ./gaussian_client.py --config config/gaussian_config.json download-url <task_id>
   ```

## Submission model
This skill targets the unified API described in `/workspace/inputs/README.md`:

- `POST /api/jobs`
- `GET /api/jobs?id=<job_id>`
- `GET /api/tasks/download?taskId=<task_id>`

It submits with:
- `software = "GAUSSIAN"`
- `cmd = "g16 {input_gjf} && formchk {input_chk}"`

For the unified API, `input_gjf` refers to the `.gjf` filename *inside the uploaded ZIP* (the remote side unzips automatically). The helper therefore supports:
- `--input-gjf-name benzene.gjf` to render the command correctly for ZIP uploads
- `input_chk = benzene.chk` from the same stem

That preserves matching filenames so the remote `formchk` step can convert the generated `.chk` into `.fchk`.

## Config reference (`scripts/config/gaussian_config.json`)
| Field | Description |
| --- | --- |
| `api_base_url` | Base URL of the unified HPC API. |
| `submit_endpoint` | Override for POST submit endpoint; defaults to `<api_base_url>/api/jobs`. |
| `status_endpoint` | Override for GET status endpoint; defaults to `<api_base_url>/api/jobs`. |
| `download_endpoint` | Override for ZIP download endpoint; defaults to `<api_base_url>/api/tasks/download`. |
| `image` | Container image passed to the HPC API. |
| `created_by` | Value used for `createdBy`. |
| `software` | Should stay `GAUSSIAN` unless the API changes again. |
| `default_cmd` | Default remote command; should remain `g16 {input_gjf} && formchk {input_chk}`. |
| `mem` / `tasks_per_node` / `n_nodes` | Default resource request. |
| `poll_interval_seconds` / `max_poll_attempts` | Poll loop controls. |

## Script details
- **Path:** `scripts/gaussian_client.py`
- **Class:** `GaussianClient`
- **CLI subcommands:**
  - `submit FILE_URL [--job-type opt|plus|minus|tdsp] [--tasks-per-node N] [--nodes M] [--mem SIZE] [--cmd '...'] [--input-gjf-name NAME.gjf]`
  - `poll JOB_ID`
  - `download-url TASK_ID`

## Example payload
```json
{
  "params": {
    "inputfile": "http://example.com/benzene.gjf",
    "software": "GAUSSIAN",
    "cmd": "g16 benzene.gjf && formchk benzene.chk",
    "n_nodes": 1,
    "tasks_per_node": 64,
    "mem": "32GB"
  },
  "image": "114.214.255.82:18080/internal/hpc-calc:latest.arm",
  "parentId": "<task-uuid>",
  "taskId": "<task-uuid>",
  "createdBy": "openclaw"
}
```

## Tips
- The submitted URL must be directly reachable by the HPC API; if needed, keep using your upload service separately.
- `job_id` is an integer used for polling; `task_id` is the UUID used for downloading results.
- If you override `cmd`, keep the `.gjf` / `.chk` stem consistent or Gaussian-to-formchk chaining will break.
