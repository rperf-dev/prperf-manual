# Setup

Getting prperf running takes about 10–15 minutes. For an overview of the
service, see "What prperf is."

There are two things to do:

1. Provide a benchmark
2. Add a workflow that runs it (triggered on both PRs and pushes to the default branch)

For a **public** repository there is nothing to install — the action itself
writes the Check Run and the sticky PR comment with the workflow's
`GITHUB_TOKEN`. **Private** repositories install the prperf GitHub App as well
(see below).

## Prerequisites

- **rperf 0.10 or newer** in your Gemfile (Bundler projects). prperf uses the
  `meta` / `summary` embedded in the profile; with an older rperf the action
  stops with a clear error. The compatibility contract is the profile's
  **format_version**, not the rperf gem version: any rperf whose profile format
  the server understands is accepted, and a too-new format is rejected with a
  clear message rather than silently misread.
- A **benchmark command** to measure (see below).
- A **public repository**. Private repositories require a paid plan (currently
  public-only during the free beta).

## Install the GitHub App (private repositories only)

**Public repositories need no install.** The action writes the Check Run and the
sticky comment directly using the workflow's `GITHUB_TOKEN`, so just add the
workflow below and you're done.

**Private repositories** install the prperf GitHub App on the repository from its
App page (a paid plan). With the App installed, prperf writes the branded Check
Run server-side. During the free beta only public repositories are supported;
private support arrives with paid plans.

## Provide a benchmark

What you measure determines the range of regressions you can catch. A good
benchmark is deterministic, runs the path you care about, and does enough work
to be stable. What and how to measure depends on your project, so the how-to and
per-project examples are in "Writing a benchmark."

For this guide we measure one concrete example: a Rails app's boot. The
benchmark is just `bin/rails runner ""` — it boots the app and runs an empty
script, so there's no benchmark file to write. The next section puts it in the
workflow; the action wraps it in rperf for you.

## Add the workflow

Add a workflow that runs your benchmark. prperf compares the **PR
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
      contents: read         # checkout
      id-token: write        # OIDC upload (no secrets)
      checks: write          # write the Check Run (public repos)
      pull-requests: write   # write the sticky comment (public repos)
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bin/rails runner ""
```

In `run:`, write just the command you want to measure — the action wraps it in
`rperf record` for you. (If you'd rather control the recording yourself, set
`record: false` and write the full `rperf record … -- <cmd>` command in `run:`.)
For a Bundler project the action runs `bundle exec rperf` so the CLI matches the
version in your Gemfile; for a project without a Gemfile the action installs
rperf itself.

You **must** include `permissions: id-token: write`. Without it there is no
OIDC token and the upload cannot happen. `contents: read` lets
`actions/checkout` fetch the repository. On a **public** repository `checks:
write` and `pull-requests: write` let the action write the Check Run and the
sticky comment with the workflow token; they are harmless on private
repositories, where the App writes the Check Run server-side. Once you set
`permissions:`, anything you don't list defaults to none, so they are all spelled
out.

## Thresholds and comments (optional)

A threshold is what gives you a ⚠️ on the Check Run when something regresses, and
where you draw the line — how much of an increase counts — is yours to set, per
metric. Concretely, it caps how much a metric (allocations, GC, time, etc.) may
increase from base to head; crossing it adds the ⚠️ and, per `comment`, a PR
comment. Thresholds are **optional**: without them the Check Run still shows
numbers, but no ⚠️ and no comment. Add them only when you want to be warned on a
regression.

All configuration lives **in the workflow** — there is no separate config file.
Write the **global defaults** once in the job's `env`, and **override per
benchmark** if needed.

```yaml
jobs:
  bench:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write, checks: write, pull-requests: write }
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
          run: bin/rails runner ""
```

Threshold keys, with a recommended starting value for each (prperf has no
built-in threshold — they take effect only once you set them):

| Key | Recommended default | Meaning |
|---|---|---|
| `alloc` | `"+10%"` | Allocation increase. Can also be absolute, e.g. `"+5000"` |
| `gc_count` | `"+2"` | GC count (minor+major) increase |
| `total_ms` | `"+20%"` | Wall-time increase. Noisy, so use relative (%) |
| `cpu_ms` | `"+15%"` | CPU-time increase |
| `method` | (none) | When a named method's self-time share exceeds the given %. E.g. `{ "JSON.generate": "15%" }` |

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

## Action inputs

| Input | Default | Description |
|---|---|---|
| `run` | (required) | Measurement command; the action wraps it in `rperf record` |
| `record` | `true` | Set `false` to write the full `rperf record … -- <cmd>` command in `run:` yourself |
| `prepare_run` | `""` | One-time setup before measuring (generate fixtures, seed, etc.); not measured |
| `count` | `3` | Number of runs; the server compares the median |
| `benchmark` | `default` | Benchmark series name; one commit can carry several, compared independently |
| `thresholds` | `""` | Thresholds for this benchmark (overrides the global defaults per key) |
| `comment` | `on_threshold` | Comment behavior |
| `server` | `https://prperf.atdot.net` | prperf server (replaceable) |
| `upload` | `true` | Set `false` to measure without uploading |

## Multiple benchmarks

You can measure one commit with several benchmarks — use **one step per
benchmark** with a distinct `benchmark` name. The server compares each against
its own base and shows them all in **one Check Run**.

```yaml
- uses: rperf-dev/prperf-action@v1
  with:
    benchmark: boot
    run: bin/rails runner ""
- uses: rperf-dev/prperf-action@v1
  with:
    benchmark: render
    run: ruby bench/render.rb
```

Use the **same benchmark names** on the PR and push-to-default-branch triggers
so each series has a baseline.

## Verify it works

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
