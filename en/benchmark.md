# Writing a benchmark

The numbers prperf reports are decided almost entirely by what you measure.
Benchmark design is the part of this chapter that most shapes the result, and
the part that takes the most effort.

## What a benchmark is

To prperf, a benchmark is the command you pass to `run:`. Usually it's a small
Ruby script (for example `bench/main.rb`) wrapped in `rperf record`:

```yaml
run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

Put rperf in your Gemfile (0.10 or newer, so `bundle exec rperf` resolves). The
action runs this `count` times (default 3) and the server compares the median
against base. What you write is the body of `bench/main.rb` — a script that does
a representative chunk of work.

## What makes a good benchmark

A good benchmark satisfies three things.

1. **It exercises the code you care about.** A PR that doesn't touch that code
   leaves the numbers unchanged.
2. **It is deterministic** (does exactly the same work every time). Otherwise
   alloc and GC drift and you get warnings that aren't regressions.
3. **It does a fixed amount of work.** If it finishes in an instant it collects
   few samples and the result is unstable.

The axis of regression judgement is the allocation count. When a benchmark is
deterministic, the allocation count is stable to the single object across PRs,
so even a small increase is caught. GC counts are deterministic and easy to
count too, so they are shown alongside. A benchmark that drifts loses that
stability.

## How to write one

The basic shape of `bench/main.rb` is: build a fixed input once, warm up, then
repeat the real loop enough times.

```ruby
# bench/main.rb
require "json"
require_relative "../config/environment"   # if needed (Rails, etc.)

# 1) Build the fixed input once (no randomness, time, or network)
DATA = { "users" => Array.new(100) { |i| { "id" => i, "name" => "user#{i}" } } }

# 2) Warm up (exclude one-time lazy loading / initialization from the measurement)
JSON.generate(DATA)

# 3) The real thing: repeat enough times
5_000.times do
  JSON.generate(DATA)
end
```

Fix the input; don't let it depend on `rand`, `Time.now`, a DB, an external API,
or filesystem enumeration order. If you truly need randomness, pin it with
`srand(42)`. Warm up so that one-time work (autoload, constant init, lazy
loading) stays out of the measurement. Tune the count (5,000 here) so the whole
run takes a few hundred ms to a few seconds: too short is unstable, too long
slows CI down.

## Check that it's deterministic

Before wiring it into CI, run it two or three times locally and confirm the
allocation and GC counts are identical each time. `rperf stat` prints the
summary to stderr.

```sh
bundle exec rperf stat -- ruby bench/main.rb
bundle exec rperf stat -- ruby bench/main.rb
```

If `allocated_objects` and the GC counts match across runs, the benchmark is
deterministic. If they drift, eliminate the causes one at a time.

- [ ] No `rand` or `SecureRandom` (pin with `srand` if you must)
- [ ] The result doesn't depend on `Time.now` or `Date.today`
- [ ] No network or external services
- [ ] No dependence on changing DB state (use fixed in-memory data, fixtures, or
      a fixed seed)
- [ ] No dependence on file enumeration order (`Dir.glob` order, etc.)
- [ ] The input size is the same every run

To get a sense of the cause, look inside with the flamegraph.

```sh
bundle exec rperf record -o out.json.gz -- ruby bench/main.rb
bundle exec rperf report out.json.gz       # opens the viewer
```

Once you've pinned down the nondeterminism, confirm the match with `rperf stat`
again before putting it into CI.

## Preparation (optional)

If you have setup that should run once before the benchmark, put it in
`prepare_run:`. Generating fixtures, seeding a DB, and building assets all
qualify. It runs once before the measurement and is not included in it.

```yaml
- uses: rperf-dev/prperf-action@v1
  with:
    prepare_run: bin/rails db:prepare db:seed   # once, before measuring
    run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/request.rb
```

A failure here fails the step. Use a fixed seed or input so each run starts from
the same state.

## Per-project examples

Use the workflow from "Setup" as is, and just point `run:` at each
`bench/*.rb`. Only benchmarks that need preparation (generating fixtures,
seeding a DB, building assets) add a `prepare_run:`.

### gem / library

Call the public API N times on deterministic, fixed input. With the input and
call count fixed, you measure only regressions in the public API.

```ruby
# bench/main.rb
require "your_gem"

# Build the fixed input deterministically (no randomness, time, or network)
DATA = { "items" => Array.new(200) { |i| { "id" => i, "name" => "item-#{i}" } } }

YourGem.encode(DATA)                 # warm up
5_000.times { YourGem.encode(DATA) }
```

Change the `require`, the fixed input `DATA`, the API you call, and the count.
You can reuse an existing `benchmark/` script (benchmark-ips and the like), but a
time-based loop varies its iteration count and makes alloc drift, so switch to a
fixed-count loop.

### Sinatra / Rack apps

For any Rack app, send one request through the full stack N times.

```ruby
# bench/request.rb
require_relative "../app"            # load your Sinatra/Rack app
require "rack/mock"

app  = Sinatra::Application           # classic style. Modular: app = MyApp
PATH = ENV.fetch("BENCH_PATH", "/")   # change to the path you want to measure
make = -> { Rack::MockRequest.env_for(PATH, "HTTP_HOST" => "localhost") }
pump = ->(r) { b = r[2]; b.each { |_| }; b.close if b.respond_to?(:close) }

3.times    { pump.call(app.call(make.call)) }   # warm up
2_000.times { pump.call(app.call(make.call)) }
```

Set `run:` to `ruby bench/request.rb`. Change the load line, `app` (the object
you `run` in `config.ru`), `PATH`, and the count. If the request reads a DB, add
a seed to the preparation and a postgres service to the workflow (see the "Rails
quickstart").

### CLI / plain Ruby

Calling the entry point in-process N times with fixed arguments avoids startup
cost and external state, and keeps samples stable.

```ruby
# bench/main.rb
require_relative "../lib/my_cli"

ARGS = %w[build --format json]        # fixed arguments
200.times { MyCli.run(ARGS) }         # call your entry point
```

To measure the executable itself, confirm it does enough work and then pass it
to `run:` directly. A single short startup collects few samples and is unstable,
so loop over it, or use the in-process loop above.

### Rails apps

Rails is covered in the next chapter, the "Rails quickstart" — boot, endpoints,
typical queries, and jobs are all there. Roda and grape are Rack apps, so measure
them as in
"Sinatra / Rack"; Hanami follows the same idea as Rails.

## Anti-patterns

The following ways of measuring move the numbers for reasons unrelated to the
PR's changes, so avoid them.

- **Measuring the test suite as is** (`rperf record -- rspec`). Adding tests in a
  PR inflates alloc, and you can't tell that apart from a regression.
- **Depending on randomness, time, or network.** It drifts every run and causes
  false positives.
- **Too short.** Time drifts and alloc is too small for a delta to show.
- **Measuring a path you barely care about.** The PR never touches it, so it
  always reads "no change."
- **Using real external dependencies (API or DB).** It drifts with the network.

A giant everything-in-one benchmark is also worth avoiding, because it's hard to
tell what regressed. Split by concern and measure several benchmarks for one
commit, and you can trace the regressed code through the Check Run and the diff
(for how to split, see "Multiple benchmarks" in "Setup").

## How this ties to thresholds

A deterministic benchmark lets you set tight relative thresholds (for example
`alloc: "+5%"`) without false positives. A benchmark that drifts forces you to
loosen the thresholds, and the signal weakens. Start with one benchmark that
measures, deterministically, the path your PRs are most likely to touch, and get
to a state where base and head show a difference.
