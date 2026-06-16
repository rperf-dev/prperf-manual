# Reading the results

prperf reports in three places: the **Check Run** (numbers), the **PR comment**
(when a threshold is exceeded), and the **flamegraph viewer** (to dig in). Here
is how to read each.

## The Check Run

A Check Run named `prperf` appears in the PR's Checks. Its **conclusion is
always success** — prperf never fails your CI. The verdict is informational.

### Title

```
2,001ms → 2,140ms (+7%) · alloc 48,741 → 59,950 (+23%) · GC 4 → 7
```

The key `base → head` metrics. If any metric exceeded its threshold, a **⚠️** is
prepended.

### Summary (body)

- **base / head metric table** — alloc, GC (minor/major), GC time, total ms,
  CPU ms, max RSS. Rows over threshold are **bold**.
- **Top 10 method diff rows** — `base self% → head self% → Δpt`, so you can see
  which methods grew in share.
- **A diff link to the viewer** — the way into the flamegraph.
- When there is no base, it shows "No base snapshot found — showing this run's
  numbers only."

### Which numbers to trust

- **Allocation and GC counts are deterministic** — they barely move even when
  the CI runner changes. These are the primary signal for regressions.
- **Time (total_ms / cpu_ms) is noisy** — CI wall time swings ±10–20%. We
  compare the **median** of `count` (default 3) runs, but treat time as
  meaningful only when it moves a lot.

## The PR comment (sticky)

When a threshold is exceeded (subject to the `comment` setting), one comment is
posted on the PR.

- **One comment per PR.** Each push **edits the same comment**, so notifications
  stay at one and the thread isn't spammed.
- Exceeded metrics are listed like
  `⚠️ **alloc** 48,741 → 59,950 (threshold +10%)`. With multiple benchmarks the
  benchmark name is included.

With `comment: never` there is no comment, only the Check Run; with `always` it
comments every time even without an exceedance.

## The flamegraph viewer

Open it from the Check Run's diff link or a shareable URL
(`/view/<repo>/<sha>`). It is rperf's viewer, so the controls are the same.

- **Flamegraph** — width is time (weight); wider is heavier.
- **Diff mode** — colors the difference between base and head: **methods whose
  share increased are red, decreased are blue**. The Check Run's diff link opens
  in this mode.
- **Time-travel sidebar** — past snapshots of that benchmark series are listed,
  so you can walk main's trend. `j` / `k` move to newer / older.
- **Pin a method** — Shift+click a method to pin it; a sparkline shows its share
  across snapshots.

Share the permanent link (`/view/<repo>/<sha>`). Under the hood the viewer
fetches a short-lived signed URL on demand, so revoked access takes effect
within ten minutes.

### Diff for a specific benchmark

Open `/view/<repo>/diff?base=<sha>&head=<sha>&bench=<name>` to diff a particular
benchmark. The viewer sidebar is scoped to one benchmark series, so boot and
endpoint snapshots never interleave.

## The dashboard

- **`/` (top)** — the marketing/explanation page.
- **`/me` (after sign-in)** — a list of repositories you can see, each linking
  to the latest result per benchmark. Public repositories appear for everyone;
  private ones only for people with GitHub read access.

Sign-in is GitHub OAuth; there is no prperf-specific account. Authorization
always follows GitHub permissions (public snapshots are viewable without
signing in).

## What to do when a regression is flagged

1. From the **⚠️ title and metric table**, see what grew (alloc / GC / time / a
   specific method) and by how much.
2. Open the flamegraph via the **diff link** and find the methods that turned
   red.
3. For an allocation increase, look for where extra objects are created; for GC,
   that's the consequence; for time, suspect **CI noise** first.
4. Push a fix and the same Check Run and sticky comment **update in place**.

## Common states

- **"No base snapshot found"** — there is no latest base-branch snapshot yet.
  The push-to-main workflow must have run at least once on a commit that is an
  ancestor of the PR.
- **Always "no change"** — the PR simply doesn't touch the path the benchmark
  exercises. Check that the benchmark covers what you care about.
- **Numbers swing every time** — the benchmark has nondeterminism (randomness,
  time, I/O). Make it deterministic, or loosen / drop the time thresholds.
