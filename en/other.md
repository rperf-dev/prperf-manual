# Quickstart for other projects

Non-Rails projects start almost as easily. Here are three: a **gem / library**,
a **Sinatra / Rack app**, and a **CLI / plain Ruby**. Each just needs one
`bench/*.rb` and `run:` pointed at it — no database, no special environment.

## The shared workflow

Non-Rails projects usually need just this minimal workflow (only `run:`
changes). PR workflow `.github/workflows/prperf.yml`:

```yaml
name: prperf
on: pull_request

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    env:
      PRPERF_DEFAULT_THRESHOLDS: |
        alloc: "+10%"
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

Add an `on: push` / `branches: [main]` version with the same steps for the base
(see Setup). Put rperf 0.10 or newer in your Gemfile.

## A gem / library

Call the public API on deterministic, fixed input N times.

```ruby
# bench/main.rb
require "your_gem"

# Build fixed input deterministically (no randomness, time, or network)
DATA = { "items" => Array.new(200) { |i| { "id" => i, "name" => "item-#{i}" } } }

YourGem.encode(DATA)                 # warm up
5_000.times { YourGem.encode(DATA) }
```

**Change**: the `require`, `DATA` (fixed input), the API you call, the count.

> You can adapt an existing `benchmark/` script (e.g. benchmark-ips), but for
> prperf use a **fixed-iteration loop**. A time-based loop varies the iteration
> count, which makes alloc jiggle.

## A Sinatra / Rack app

Any Rack app: send one request through the full stack N times.

```ruby
# bench/request.rb
require_relative "../app"            # load your Sinatra/Rack app
require "rack/mock"

app  = Sinatra::Application           # classic style. Modular: app = MyApp
PATH = ENV.fetch("BENCH_PATH", "/")   # ← change to the path you care about
make = -> { Rack::MockRequest.env_for(PATH, "HTTP_HOST" => "localhost") }
pump = ->(r) { b = r[2]; b.each { |_| }; b.close if b.respond_to?(:close) }

3.times    { pump.call(app.call(make.call)) }   # warm up
2_000.times { pump.call(app.call(make.call)) }
```

Point the workflow's `run:` at `ruby bench/request.rb`.

**Change**: the load line and `app` (the object you `run` in `config.ru`),
`PATH`, the count. If the request reads a DB, add a postgres service and seed to
the shared workflow (see Rails quickstart ②).

## A CLI / plain Ruby

Calling the entry point **in-process** with fixed arguments N times gives the
most stable samples.

```ruby
# bench/main.rb
require_relative "../lib/my_cli"

ARGS = %w[build --format json]        # fixed arguments
200.times { MyCli.run(ARGS) }         # call your entry point
```

To measure the executable itself, first make sure it does enough work:

```yaml
run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby exe/mycli build fixtures/sample.txt
```

But a single short invocation collects few samples and is unstable. **Loop**
(process a large fixed input inside), or use the in-process loop above.

## Other frameworks

- **Hanami / Roda / grape, etc.** — Roda and grape are Rack apps, so use the
  "Sinatra / Rack" approach above. Hanami follows the same idea as Rails: measure
  boot via the app's boot, and a request through Rack.

When in doubt, start with one deterministic benchmark of your most important
path. See "Writing a benchmark" for the details.
