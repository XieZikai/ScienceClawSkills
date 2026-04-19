# Remote Job Orchestrator Templates (SQLite + systemd user)

## 1. Submission Payload (map to `submit` CLI)
Use one record per remote task.

```yaml
workflow: gaussian16-opt
remote_job_id: "123456"
poll_command: "python3 tools/check_gaussian_status.py --job 123456"
fetch_command: "python3 tools/fetch_gaussian_outputs.py --job 123456"
continue_command: "python3 tools/run_post_analysis.py --job 123456"
callback_command: "python3 tools/on_state_change.py"
check_every: 300
poll_timeout: 120
max_retries: 80
context:
  done_regex: "FINISHED|NORMAL TERMINATION"
  failed_regex: "FAILED|ERROR|CANCELLED"
  running_regex: "RUNNING|QUEUED|PENDING"
```

## 2. Callback Contract
`callback_command` receives environment variables:

- `ORCH_JOB_ID`
- `ORCH_WORKFLOW`
- `ORCH_REMOTE_JOB_ID`
- `ORCH_OLD_STATUS`
- `ORCH_NEW_STATUS`
- `ORCH_NOTE`

Recommended callback behavior:
1. Write state change to your own event bus/chat notifier.
2. If `ORCH_NEW_STATUS=completed`, enqueue dependent step instead of assuming original TUI session exists.
3. Never block longer than 60s.

## 3. Poll Script Output Contract
Your poll script should print one of these words (or your own regex targets):

- Complete: `FINISHED`, `DONE`, `SUCCESS`
- Failed: `FAILED`, `ERROR`, `CANCELLED`
- Running: `RUNNING`, `PENDING`, `QUEUED`

If output is ambiguous:
- Return code `0` means running.
- Nonzero/timeout means retry.

## 4. Health Check Commands
```bash
# Inspect timer
systemctl --user status openclaw-orchestrator.timer

# Force one immediate tick
systemctl --user start openclaw-orchestrator.service

# Recent jobs
python3 scripts/persistent_worker.py --db "$HOME/.local/share/openclaw-orchestrator/orchestrator.db" list --limit 20

# Debug one job with event log
python3 scripts/persistent_worker.py --db "$HOME/.local/share/openclaw-orchestrator/orchestrator.db" show <job_id> --events 50
```

## 5. Minimal Failure Playbook
1. `timer inactive`: run bootstrap again, then `systemctl --user daemon-reload && systemctl --user enable --now openclaw-orchestrator.timer`.
2. `job stuck in retry`: inspect `show <job_id>`, fix `poll_command`/regex, then resubmit.
3. `callback fails`: callback errors are in the `events` table (`event_type=callback_error`).
