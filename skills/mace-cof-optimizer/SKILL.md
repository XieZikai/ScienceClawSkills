---
name: mace-cof-optimizer
description: Submit, poll, and download MACE-optimized COF CIFs via the remote scorer API. Use when you need to batch CIF files, send them to the MACE pipeline, and retrieve the optimized structures/results.
---

# MACE COF Optimizer Skill

## Quick start
1. **Prepare config**
   - Copy `scripts/config/mace_optimizer_config.example.json` → `scripts/config/mace_optimizer_config.json`.
   - Set `scorer_url` to the Cloudflare tunnel (or your own gateway). Optional fields allow overriding endpoints/timeouts.
2. **Collect CIFs**
   - Place the starting COF `.cif` files in a folder. Filenames are sorted before submission; non-CIF files are ignored.
3. **Submit the job**
   ```bash
   cd skills/mace-cof-optimizer/scripts
   ./mace_cof_client.py --config config/mace_optimizer_config.json \
     submit /path/to/cif_folder experiment_20260322
   ```
   - This wraps `get_opt_only_input_json`: the JSON payload contains the combined CIF text plus the default `optimization_conf` (MACE off, GPU config, etc.).
4. **Poll until finished**
   ```bash
   ./mace_cof_client.py --config config/mace_optimizer_config.json poll <job_id>
   ```
   - Uses the configured interval/timeout; resilient to transient HTTP errors.
5. **Fetch results / CIFs**
   ```bash
   ./mace_cof_client.py --config config/mace_optimizer_config.json result --job-id <job_id>
   ./mace_cof_client.py --config config/mace_optimizer_config.json cifs --job-id <job_id> --out outputs/mace_results
   ```
   - The `cifs` subcommand downloads the JSON payload and (optionally) writes `structure_###.cif` files to disk.

## CLI reference
| Command | Purpose |
| --- | --- |
| `submit FOLDER NAME` | Uploads every `.cif` in `FOLDER` (sorted) and starts an optimization job named `NAME`. Prints the server response containing `job_id`. |
| `poll JOB_ID` | Blocks until the job finishes or hits the configured timeout. |
| `result [--job-id | --job-name]` | Fetches the metadata JSON from `/result`. |
| `cifs [--job-id | --job-name] [--out DIR]` | Fetches `/result_cifs`. If `--out` is set, saves each CIF as `structure_###.cif`. |

All commands accept `--config /path/to/mace_optimizer_config.json`; if omitted, the default path under `scripts/config/` is used.

## Config reference (`scripts/config/mace_optimizer_config.json`)
| Field | Description |
| --- | --- |
| `scorer_url` | Base URL to the scorer service (required). |
| `submit_url` / `poll_url` / `result_url` / `result_cifs_url` | Optional explicit endpoints if they differ from the defaults (`<base>/submit`, `<base>/status`, etc.). |
| `poll_interval_seconds` | Delay between status checks (default 100s). |
| `poll_timeout_seconds` | Total time before `poll` raises `TimeoutError`. |
| `request_timeout_seconds` | HTTP timeout for GET requests. |

## Implementation notes
- `cifs_from_folder` and `get_opt_only_input_json` mirror your original helper functions; no schema changes required.
- `submit_task`, `poll_status`, `get_result`, and `get_result_cifs` are wrapped inside `MACEOptimizerClient`, so they can be imported into other automation scripts or used via CLI.
- The CLI prints all JSON responses, making it easy to capture provenance logs or feed downstream workflows.
