# Runner Lag Chaos Harness

This harness dispatches jobs onto one MonkCI runner pool and deliberately creates:

- cancellation requested immediately after dispatch,
- cancellation requested only after runner assignment,
- concurrency replacement,
- fast failure after assignment,
- clean baseline jobs,
- capacity-holding jobs that create queue pressure.

It never cancels workflows outside the uniquely named experiment and it does not deliberately exhaust VM memory.

## Install

Copy the two workflow files and script into a repository whose workflows use MonkCI runners:

```text
.github/workflows/runner-lag-chaos.yml
.github/workflows/runner-lag-chaos-target.yml
scripts/runner-lag-chaos.sh
```

Commit them to the repository's default branch. The target must be known to GitHub before `workflow_dispatch` can dispatch it.

The orchestrator uses `GITHUB_TOKEN` with `actions: write`. If organization policy prevents dispatch/cancel using that token, add an Actions-write token as the repository secret `CHAOS_GH_TOKEN`.

## Run

```bash
gh workflow run runner-lag-chaos.yml \
  --ref main \
  -f confirm=RUNNER-LAG-CHAOS \
  -f runner_label=monkci-ubuntu-24.04-4 \
  -f run_count=24 \
  -f seed=20260804 \
  -f cancel_queued_percent=20 \
  -f cancel_running_percent=25 \
  -f concurrency_percent=20 \
  -f fail_percent=10 \
  -f hold_seconds=180 \
  -f observation_minutes=15
```

The artifact contains:

- `manifest.jsonl`: intended scenario and child run IDs,
- `results.csv`: queue time, runner, conclusion and anomaly classification,
- `results.json`: complete structured summary,
- `raw/`: GitHub Actions run/job API responses,
- `summary.md`: quick analysis,
- `logs-explorer-filter.txt`: starting filter containing the relevant GitHub run IDs.

## First run recommendation

Use a non-production test repository and `monkci-ubuntu-24.04-4` with 12 runs. Then repeat the exact same seed with 24 and 40 runs. A deterministic seed lets you compare controller versions without changing the chaos sequence.
