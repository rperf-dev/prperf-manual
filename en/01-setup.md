# Setup

Getting prperf running takes about 10–15 minutes. For an overview of the
service, see "What prperf is."

There are three things to do:

1. Install the GitHub App
2. Provide a benchmark
3. Add a workflow that runs it (triggered on both PRs and pushes to the default branch)

## 1. Prerequisites

- **rperf 0.10 or newer** in your Gemfile. prperf uses the `meta` / `summary`
  embedded in the profile; with an older rperf the action stops with a clear
  error.
- A **benchmark command** to measure (see below).
- A **public repository**. Private repositories require a paid plan (currently
  public-only during the free beta).

## 2. Install the GitHub App

Install the prperf GitHub App on your repository from its App page. This lets
prperf write the Check Run and PR comments for that repository.

## 3. Provide a benchmark

What you measure determines the range of regressions you can catch. A good
benchmark is deterministic, runs the path you care about, and does enough work
to be stable. The full how-to and per-project examples are in "Writing a
benchmark" — start from the template there if you don't have one yet.

## 4. Add the workflow

Add a workflow that runs the benchmark from step 3. prperf compares the **PR
head** against the **latest snapshot of the base branch**. Trigger the workflow
on both `push` (the default branch) and `pull_request`: the push records the
base, and the PR is compared against it (prperf tells them apart from the OIDC
token's ref). List both `main` and `master` for the branch so it works either
way — only your default branch exists, so only that one fires. Until the default
branch has been pushed once, there's nothing to compare against: "No base
snapshot found — showing this run's numbers only."

```yaml
# .github/workflows/prperf.yml
name: prperf
on:
  push:
    branches: [ main, master ]   # records the base (your default branch)
  pull_request:                  # compared against the base

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read         # for checkout
      id-token: write        # required for OIDC upload
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

`run:` must make rperf write at least one profile. Point its output at the
action-provided `$PRPERF_DIR` with `--snapshot-dir "$PRPERF_DIR"`.

You **must** include `permissions: id-token: write`. Without it there is no
OIDC token and the upload cannot happen. `contents: read` lets
`actions/checkout` fetch the repository; once you set `permissions:`, anything
you don't list defaults to none, so both are spelled out.

## 5. Thresholds and comments (optional)

Thresholds are **optional**. Without them the Check Run still shows numbers, but
no ⚠️ and no comment. Add them only when you want to be warned on a regression.

All configuration lives **in the workflow** — there is no separate config file.
Write the **global defaults** once in the job's `env`, and **override per
benchmark** if needed.

```yaml
jobs:
  bench:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write }
    env:
      PRPERF_DEFAULT_THRESHOLDS: |     # applies to every benchmark
        alloc: "+10%"
        total_ms: "+20%"
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with: { bundler-cache: true }
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

Threshold keys:

| Key | Example | Meaning |
|---|---|---|
| `alloc` | `"+10%"` / `"+5000"` | Allocation increase (relative / absolute) |
| `gc_count` | `"+2"` | GC count (minor+major) increase |
| `total_ms` | `"+20%"` | Wall time (noisy; prefer relative) |
| `cpu_ms` | `"+15%"` | CPU time |
| `method` | `{ "JSON.generate": "15%" }` | Method self-time share exceeding an absolute value |

- Summary values are `"+N%"` (relative) or `"+N"` (absolute); method values are
  `"N%"`.
- Invalid values are ignored, with one warning line on the Check Run (CI is
  never failed).
- Relative thresholds (`+10%`) generalize cleanly across benchmarks. Absolute
  and method thresholds mean different things per benchmark, so override them
  per benchmark only when needed.

Comment behavior is controlled by the `comment` input (default `on_threshold`):

| Value | Behavior |
|---|---|
| `on_threshold` | Comment only when a threshold is exceeded (default) |
| `always` | Comment every time |
| `never` | Never comment (Check Run only) |

There is one comment per PR, and each push **edits the same comment**, so
notifications stay at one.

## 6. Action inputs

| Input | Default | Description |
|---|---|---|
| `run` | (required) | Measurement command; must emit at least one `.json.gz` |
| `prepare_run` | `""` | One-time setup before measuring (generate fixtures, seed, etc.); not measured |
| `count` | `3` | Number of runs; the server compares the median |
| `benchmark` | `default` | Benchmark series name; one commit can carry several, compared independently |
| `thresholds` | `""` | Thresholds for this benchmark (overrides the global defaults per key) |
| `comment` | `on_threshold` | Comment behavior |
| `server` | `https://prperf.atdot.net` | prperf server (replaceable) |
| `upload` | `true` | Set `false` to measure without uploading |

## 7. Multiple benchmarks

You can measure one commit with several benchmarks — use **one step per
benchmark** with a distinct `benchmark` name. The server compares each against
its own base and shows them all in **one Check Run**.

```yaml
- uses: rperf-dev/prperf-action@v1
  with:
    benchmark: boot
    run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- bin/rails runner ""
- uses: rperf-dev/prperf-action@v1
  with:
    benchmark: render
    run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/render.rb
```

Use the **same benchmark names** on the PR and push-to-default-branch triggers
so each series has a baseline.

## 8. Verify it works

1. Push to main first → the workflow runs on the push and the base snapshot
   reaches the server.
2. Open a PR → the workflow runs on the PR; **numbers on the Check Run** mean
   success.
3. A link to the uploaded result also appears in each job's **Summary**.

## Limitations

- **PRs from forks cannot upload.** GitHub does not grant `id-token: write` to
  fork-triggered workflows, so no OIDC token is available. Same-repository
  branch PRs work normally.
- Upload problems (plan limits, rate limits, server errors) are **warnings
  only**; the step still succeeds. Only the measurement command itself failing
  fails the step.
- During the free beta, **public repositories only**. Private repositories are
  coming with paid plans.
