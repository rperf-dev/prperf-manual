# Writing a benchmark

prperf's value is decided almost entirely by **what you measure**. This is the
most important — and most effort-intensive — part. Here is how to write a good
benchmark, with concrete examples.

## What a benchmark is (in prperf)

The command you pass to `run:` is the "benchmark." Usually it's a **small Ruby
script** (e.g. `bench/main.rb`) wrapped in `rperf record`:

```yaml
run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

The action runs it `count` (default 3) times and the server compares the median
against base. What you write is the body of `bench/main.rb` — **a script that
does a representative chunk of work**.

## The core principle: "the path you care about, deterministically, enough of it"

A good benchmark has three properties:

1. **Exercises what you care about** — a PR that doesn't touch that code won't
   move the numbers.
2. **Deterministic** (does exactly the same work every time) — otherwise alloc
   and GC jiggle and you get warnings that aren't regressions.
3. **Does enough work** — a benchmark that finishes instantly collects few
   samples and is unstable.

prperf's primary metrics (allocation and GC counts) are deterministic, so **as
long as your benchmark is deterministic, these are stable to the single unit
across PRs**. A jiggly benchmark throws that strength away.

## Skeleton (template)

```ruby
# bench/main.rb
require "json"
require_relative "../config/environment"   # if needed (Rails, etc.)

# 1) Build fixed input once (no randomness, time, or network)
DATA = { "users" => Array.new(100) { |i| { "id" => i, "name" => "user#{i}" } } }

# 2) Warm up (exclude one-time lazy loading / initialization from the measurement)
JSON.generate(DATA)

# 3) The real thing: repeat enough times
5_000.times do
  JSON.generate(DATA)
end
```

Key points:

- **Fixed input.** Don't depend on `rand`, `Time.now`, a DB, an external API, or
  filesystem enumeration order. If you truly need randomness, pin it with
  `srand(42)`.
- **Warm up** to exclude one-time work (autoload, constant init, warmup effects)
  from the measurement.
- **The count (5,000 here)** should make the whole run a few hundred ms to a few
  seconds. Too short is unstable; too long slows CI.

## Determinism checklist

- [ ] No `rand` / `SecureRandom` (or pinned with `srand`)
- [ ] Result does not depend on `Time.now` / `Date.today`
- [ ] No network or external services
- [ ] Uses fixed in-memory data / fixtures, not a real DB
- [ ] Does not depend on file enumeration order (`Dir.glob`, etc.)
- [ ] Input size is the same every run

## Check locally that it doesn't jiggle

Before wiring it into CI, run it a few times locally and confirm the
**allocation and GC counts are identical** each time. `rperf stat` prints the
summary to stderr.

```sh
bundle exec rperf stat -- ruby bench/main.rb
bundle exec rperf stat -- ruby bench/main.rb
```

If `allocated_objects` and the GC counts match across runs, it's deterministic.
If they vary, hunt for the nondeterminism using the checklist above. To inspect
the flamegraph locally:

```sh
bundle exec rperf record -o out.json.gz -- ruby bench/main.rb
bundle exec rperf report out.json.gz       # opens the viewer
```

## What to measure (examples by project type)

### A gem / library

Call the public API on representative fixed input N times.

```ruby
require "your_lib"
doc = File.read("bench/fixtures/sample.xml")   # commit a fixed fixture
2_000.times { YourLib.parse(doc) }
```

### A Rails app

- **Boot** — the zero-config starting point. Just measuring `bin/rails runner ""`
  catches boot slowdowns from eager loading or added gems (see Setup).
- **One request** — boot the test environment and send a fixed request through
  `Rack`; hit the same endpoint N times against fixed seed data.
- **A typical query / service** — run the logic against fixed in-memory data N
  times.
- **A job** — `SomeJob.new.perform(fixed_args)` N times.

### A CLI tool

Run a representative subcommand on fixed input once (or a few times).

## Don't cram everything into one benchmark

A giant "everything" benchmark makes it hard to tell what regressed. Prefer to
**split by concern**. prperf compares **multiple benchmarks independently** for
one commit — just use separate steps with different `benchmark:` names (see
"Multiple benchmarks" in Setup).

```yaml
- uses: rperf-dev/prperf-action@v1
  with: { benchmark: parse,     run: 'bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/parse.rb' }
- uses: rperf-dev/prperf-action@v1
  with: { benchmark: serialize, run: 'bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/serialize.rb' }
```

## Anti-patterns

- **Measuring the test suite directly** (`rperf record -- rspec`). Adding tests
  in a PR inflates alloc, so you can't separate that from a regression. Doing it
  well needs normalization.
- **Depending on randomness / time / network** → jiggles, false positives.
- **Too short** → time is noisy and alloc is too small to show a delta.
- **Measuring a path you barely touch** → the PR never moves it; always "no
  change."
- **Real external dependencies (API, DB)** → varies with the network.

## How this ties to thresholds

A deterministic benchmark lets you set **tight relative thresholds** (e.g.
`alloc: "+5%"`) without false positives. A jiggly one forces loose thresholds
and a weak signal. In short: **a good benchmark means a sharp threshold.** Start
with one benchmark that measures your single most important path,
deterministically.
