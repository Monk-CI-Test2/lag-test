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

---

# Runner Assignment Stress Harness

A second, more aggressive harness that specifically tests the controller's
**receipt-based assignment reconciliation** (the 90s assigned-start timeout +
requeue fix). Where `runner-lag-chaos` fires one burst, this one sustains
pressure across multiple waves and gives a hard PASS/FAIL verdict.

## What it does differently

- **Waves timed against the recovery cycle** (default 120s ≈ 90s timeout +
  30s sweep): jobs requeued by the fix re-enter allocation exactly as a fresh
  wave of younger jobs arrives, so replacement runners are also at theft risk.
- **Late cancels** (10–25s after dispatch) land *after* the controller has
  registered a runner, freeing live runners for GitHub to hand to other jobs.
- **First-wave surge** oversubscribes the pool to guarantee backlog and
  runner permutation.
- **A final zero-chaos clean wave** detects false-positive requeues.
- **Hard verdict**: the orchestrator run FAILS if any must-run job never got a
  runner, exceeded the recovery SLO, or had to be force-cancelled. Leftover
  runs are always force-cancelled so the repository is left clean.

## Run

```bash
gh workflow run runner-assignment-stress.yml \
  --ref master \
  -f confirm=RUNNER-ASSIGNMENT-STRESS \
  -f runner_label=monkci-ubuntu-24.04-4 \
  -f waves=3 \
  -f runs_per_wave=12 \
  -f wave_interval_seconds=120 \
  -f chaos_intensity=60 \
  -f surge_factor_percent=150 \
  -f recovery_slo_seconds=300 \
  -f observation_minutes=12 \
  -f seed=0
```

Extreme preset: `waves=5 runs_per_wave=16 chaos_intensity=90
surge_factor_percent=200 wave_interval_seconds=90` (80 runs).

## Inputs

| Input | Meaning |
|---|---|
| `waves` | Chaos waves (1–6); a clean control wave is always appended |
| `runs_per_wave` | Runs per chaos wave (4–20, waves×runs ≤ 80) |
| `wave_interval_seconds` | Wave spacing; 120 aligns with one recovery cycle |
| `chaos_intensity` | Percent of each wave that is hostile (≤ 90); split ~25/25/20/15/15 across fast-cancel / late-cancel / cancel-running / concurrency-replace / fail-after-start |
| `surge_factor_percent` | First-wave size vs `runs_per_wave` (100–300) |
| `recovery_slo_seconds` | Max acceptable queue time for a must-run job (timeout + sweep + VM boot + margin) |
| `seed` | Deterministic scenario assignment; 0 = orchestrator run ID. Reuse a seed to compare controller versions |

## Reading the result

- **Green run** = every job that should run received a runner within the SLO,
  and the clean wave produced zero anomalies. The assignment ledger held.
- **Red run** = check the step summary's *Cases requiring inspection* table:
  `stranded_no_runner` / `stranded_forced_cancel` mean the pre-fix deadlock
  reappeared; `queue_over_slo` means recovery worked but too slowly.
- `estimated_recovery_cycles` in `results.json` is queue time ÷ ~130s — a
  proxy for how many requeue cycles a job needed (capacity waits inflate it).
  Cross-check real recoveries in controller logs via
  `logs-explorer-filter.txt` from the artifact.

The artifact layout matches the chaos harness (`manifest.jsonl`, `results.json`,
`verdict.json`, `raw/`, `summary.md`, `logs-explorer-filter.txt`).
