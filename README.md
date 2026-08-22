# revi-hermes-target

Target repo for [Re:vi](https://github.com/bookang869/Re-vi)'s Hermes triage agent — the repo Hermes actually patches. Not the production app; a stand-in for it, public so the pipeline has somewhere to open real PRs and real merge commits without touching anything live.

## How it fits into Re:vi

When Re:vi's Go Ingestion Gateway sees a production alert, it fires a `repository_dispatch` at this repo, which triggers [`hermes-triage.yml`](.github/workflows/hermes-triage.yml). That workflow spins up an ephemeral GitHub Actions runner, boots Hermes inside it, and lets it diagnose + patch one of the fixture apps below. Two modes, chosen by the Gateway per-alert and stamped into the dispatch payload:

- **PR_REVIEW** (default/fallback) — Hermes fixes the bug, pushes `hermes/hotfix-[alert-id]`, opens a same-repo PR against `main`, and pages a human. Never touches `main` directly.
- **AUTONOMOUS** — same repair loop, plus: boot the patched app, run a synthetic smoke test, run the app's full existing test suite, and only on a clean pass merge into `main` via the GitHub REST API (a plain, unsigned commit). Any failure at any stage aborts, drops the branch, and pages instead of merging.

Every run — pass or fail — reports its outcome back to the Gateway's digest endpoint, which rolls up into the `#triage-morning-review` Slack digest.

Full architecture, token scoping, and design rationale live in the main repo: see `Re-vi/CLAUDE.md`, `docs/TRD.md`, and `docs/PRD.md`.

## What's in here

- **`fixture-app-go/`, `fixture-app-node/`, `fixture-app-python/`, `fixture-app-rust/`** — small standalone apps, one per language, that stand in for "the production service." Faults get injected into these (see the benchmark suite in the main repo) and Hermes patches them for real, with real compilers/test runners.
- **`scripts/`** — the shell/Python glue each workflow job calls: `hermes-wrapper.sh` (runs the repair loop, gates on independent build/test verification), `resolve-boot-command.sh` (maps an alert's `service_name` to the right fixture app's boot command), `smoke-test.sh`, `autonomous-promote.sh` / `promote-pr.sh` (the merge/PR paths), `report-digest.sh` (always-run outcome reporting), plus `fake-agent.sh` / `fake-github-api.py` test doubles used by rehearsals, and a `test_*.sh` suite for the scripts themselves.
- **`.github/workflows/`**:
  - `hermes-triage.yml` — the real entry point; only ever triggered by the Gateway's `repository_dispatch`.
  - `hermes-rehearsal.yml` — a manual (`workflow_dispatch`) dry run of the same PR_REVIEW/AUTONOMOUS logic with fake inputs and test doubles, so the pipeline can be exercised without a real alert.
  - `build-hermes-image.yml` — weekly (plus manual) rebuild of the pre-baked `hermes-runner` container image pushed to GHCR, so real dispatches pay a fast image pull instead of installing Hermes + toolchains from scratch every run.
  - `revi-keepalive.yml` — weekly no-op-ish job that just touches the Hermes memory cache so GitHub's 7-day cache eviction doesn't wipe it during quiet weeks with no real alerts.
- **`Dockerfile`** — the `hermes-runner` image: Ubuntu 24.04 + Go/Rust/Node/Python toolchains + Hermes CLI, built by `build-hermes-image.yml`.

## Not in here

This repo intentionally does **not** contain the Gateway, the observability stack, or Re:vi's own docs/benchmark — those live in `bookang869/Re-vi`. This repo is only the thing Hermes is let loose on.
