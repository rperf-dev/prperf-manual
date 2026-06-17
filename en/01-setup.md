# Setup

Getting prperf running takes about 10–15 minutes. For an overview of the
service, see "What prperf is."

There are three things to do:

1. Install the GitHub App
2. Add the workflows (one for PRs, one for pushes to the default branch)
3. Provide a benchmark

## 1. Prerequisites

- **rperf 0.10 or newer** in your Gemfile. prperf uses the `meta` / `summary`
  embedded in the profile; with an older rperf the action stops with a clear
  error.
- **Ruby 3.4 or newer** (required by rperf 0.10). If your CI `ruby/setup-ruby`
  uses an older Ruby, `bundle install` (`gem install rperf`) fails.
- A **benchmark command** to measure (see below).
- A **public repository**. Private repositories require a paid plan (currently
  public-only during the free beta).

## 2. Install the GitHub App

Install the prperf GitHub App on your repository from its App page. This lets
prperf write the Check Run and PR comments for that repository.

## 3. Add the workflows

prperf compares the **PR head** against the **latest snapshot of the base
branch**, so you need **two** workflows:

- **For PRs**: measure and upload on every PR (the side being compared)
- **For pushes to main**: measure on each push to the default branch (this
  supplies the comparison **base**)

Without the main workflow there is nothing to compare against, and you get
"No base snapshot found — showing this run's numbers only."

### PR workflow

```yaml
# .github/workflows/prperf.yml
name: prperf
on: pull_request

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
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

You **must** include `permissions: id-token: write`. Without it there is no
OIDC token and the upload cannot happen.

### Push-to-main workflow

```yaml
# .github/workflows/prperf-base.yml
name: prperf (base)
on:
  push:
    branches: [ main ]

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

Keep `run:` identical in both — you can only compare like with like.

## 4. Provide a benchmark

This is the most important and the most effort-intensive part. **What you
measure** determines the value.

`run:` is any command that emits at least one rperf profile. `$PRPERF_DIR` is a
directory the action provides; passing it to `--snapshot-dir "$PRPERF_DIR"` is
the usual pattern.

A good benchmark is:

- **Deterministic** — the less it depends on randomness, time, the network, or
  external I/O, the better (allocations and GC counts stay stable).
- **On the path you care about** — a PR that doesn't touch what the benchmark
  exercises won't move the numbers, so the Check will read "no change" every
  time.
- **Reasonably sized** — a benchmark that finishes instantly collects few
  samples and is noisy.

### Zero-config start (Rails boot)

If you don't have a meaningful benchmark yet, **measuring boot** works for any
Rails app and is deterministic. It catches a real class of regression — boot
slowdowns from eager loading or added gems.

```yaml
run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- bin/rails runner ""
```

Use this to get the "numbers appear" experience first, then graduate to a real
benchmark.

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

### Comment behavior

Controlled by the `comment` input (default `on_threshold`):

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

Use the **same benchmark names** in the PR and push-to-main workflows so each
series has a baseline.

## 8. Verify it works

1. Push to main first → the main workflow runs and the base snapshot reaches the
   server.
2. Open a PR → the PR workflow runs; **numbers on the Check Run** mean success.
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
