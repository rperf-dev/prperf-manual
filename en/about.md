# What prperf is

First, let's be clear about **what prperf is**.

## In one line

**prperf is a thin GitHub App that automatically checks each PR for performance
regressions and reports the result on the PR.** Measurement happens inside your
CI with the open-source Ruby profiler [rperf](https://github.com/ko1/rperf); prperf
itself just compares **the base branch's latest measurement (usually main)**
against **this PR's**, and reports.

We call that base-branch baseline **base** and the PR side **head** — the same
terms GitHub uses for pull requests. The rest of this manual uses them.

> If you know **Codecov** for test coverage, prperf is that, for performance.

rperf is a **time (CPU) sampling profiler** — at heart a **flamegraph** of where
time went. prperf pulls **run time, GC, and allocations** out of that profile and
compares base vs head. Open a PR and the Check Run shows a **summary** like:

> 2,001ms → 2,140ms (+7%) · alloc 48,741 → 59,950 (+23%) · GC 4 → 7

prperf lets you notice "how performance changed in this commit" **at the PR
stage, before it merges**.

## What it does / doesn't do

**It does:**

- Show the base→head performance delta (allocations, GC, time) on the Check Run
  for every PR
- Comment on the PR only when a threshold is exceeded (sticky, quiet
  notifications)
- Visualize *which method got heavier* with a flamegraph diff

**It does not:**

- **It is not production monitoring.** It complements Datadog / Grafana rather
  than replacing them (those watch production; prperf catches regressions at the
  PR stage).
- **It never fails your CI.** The verdict is informational; the Check's
  conclusion is always success.
- **It never runs your code on the server.** Measurement happens **inside your
  CI**, and prperf only receives the result (the profile) and compares it. That
  is what makes it a "thin" App — light on both security and cost.

## The big picture

```
Your CI (GitHub Actions)
  └─ prperf-action
       ├─ measures your benchmark N times with rperf
       └─ uploads the profiles (.json.gz) to the prperf server
            │  (authenticated with the GitHub OIDC token — no secrets)
            ▼
prperf server
  ├─ compares base vs PR
  └─ reports on the Check Run / PR comment
```

## The user experience

1. Add a few lines of the provided GitHub Action to your workflow (a public
   repository needs nothing else; a private repository also installs the GitHub
   App)
2. Open a PR and the result appears on the Check and a PR comment

## Why it's trustworthy (the design ideas)

- **The verdict leans on deterministic metrics.** What rperf measures is mainly
  time (the flamegraph), but CI wall time swings ±10–20%. So **for the verdict**
  prperf weights the **allocation and GC counts**, which don't (time stays
  informational). The result still shows time, GC, allocations, and the
  flamegraph.
- **No secrets.** Authentication is the GitHub Actions OIDC token — no API keys
  to issue or manage.
- **Quiet notifications.** The Check Run is a permanent home (zero
  notifications); a comment appears only on a threshold breach, one per PR,
  edited in place.
- **You see "why."** The flamegraph diff pinpoints the method that got heavier.

## Who it's for

prperf suits authors of public gems and libraries who want to stop performance
regressions at the PR. A dependency bump or a refactor can quietly add
allocations or slow the boot; prperf catches it before the PR merges.

It also suits teams whose private apps care about performance — Rails apps where
speed drives UX or revenue. You catch a heavy change in the same place you review
it, before it reaches production.

Pricing: public repositories are free, private repositories are on a paid plan
(currently public-only during the free beta).

## How to read this manual

- **Setup** — installing and adding the workflows
- **Writing a benchmark** / **the Rails quickstart** — what to
  measure
- **Reading the results** / **Reading a flamegraph** — interpreting the output

Next, head to Setup.
