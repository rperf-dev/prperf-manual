# prperf Manual

**prperf** is a thin GitHub App that **checks each PR for performance
regressions**. Measurement happens inside your CI with the open-source sampling
profiler [rperf](https://github.com/ko1/rperf); prperf only compares **the base
branch's latest (e.g. main) against this PR** and reports the result on the PR.
It complements production monitoring (Datadog / Grafana) rather than competing
with it. (If you know Codecov for test coverage, this is that, for performance.)

Open a PR and the Check Run shows numbers like:

> 2,001ms → 2,140ms (+7%) · alloc 48,741 → 59,950 (+23%) · GC 4 → 7

- **No secrets** — authentication uses the GitHub Actions OIDC token
- **Never blocks CI** — the verdict is informational only
- **Deterministic metrics** — allocation and GC counts are robust to CI noise
- **Flamegraph diff** — see which method got heavier in the viewer

Start with "**What prperf is**" for the overview, then head to "Setup."
