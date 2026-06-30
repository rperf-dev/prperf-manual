# prperf Manual

prperf is a thin GitHub App that checks each PR for performance regressions. Measurement runs inside your CI via the open-source Ruby sampling profiler [rperf](https://github.com/ko1/rperf); prperf just compares the base (e.g. main) against this PR and reports on the PR. If you know Codecov for test coverage, this is that, for performance.

Open a PR and the Check Run shows numbers like:

> 2,001ms → 2,140ms (+7%) · alloc 48,741 → 59,950 (+23%) · GC 4 → 7

The next chapter, "What prperf is," gives the overview.

## A tour of prperf

### Setup

1. Provide a benchmark. Here we'll measure boot time with `bin/rails runner ""`.
2. Add a workflow that runs it, triggered on both `push` (the default branch) and `pull_request`. (A public repository needs nothing installed; a private repository also installs the GitHub App.)

```yaml
# .github/workflows/prperf.yml
name: prperf
on:
  push:
    branches: [main, master]   # records the base (default branch; list both so main or master works)
  pull_request:                # compared against the base
jobs:
  bench:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write, checks: write, pull-requests: write }
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with: { bundler-cache: true }
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bin/rails runner ""   # ← your measurement command (the action wraps it in rperf)
```

Options like threshold alerts, multiple benchmarks (`benchmark`), comment control (`comment`), and run count (`count`, default 3, median) are available too (see "Setup").

### Results

On each PR, the result shows up right in the PR's Checks (a summary compared against the base). A comment is posted only when a threshold is exceeded, and the flamegraph diff shows which method got heavier (see "Reading results").

Every PR and push also records a measurement, so you can browse the history over time at [prperf.atdot.net](https://prperf.atdot.net).

prperf never blocks CI and needs no secrets. PRs from forks can't be measured, and during the free beta only public repositories are supported.
