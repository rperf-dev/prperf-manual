# Rails quickstart

Before agonizing over "what to measure," here is an **almost copy-paste**
starting point for Rails, in two steps. Step ① gives you the "numbers appear"
experience in 30 seconds; go to ② when you want more.

## Measure boot first (no extra files)

`bin/rails runner ""` **boots the app and exits doing nothing**, so it measures
boot itself. It catches added gems, heavier initializers, and autoload changes;
it is **deterministic** and needs no extra files and no database.

Paste this as `.github/workflows/prperf.yml`:

```yaml
name: prperf
on:
  push:
    branches: [main, master]   # records the base (default branch)
  pull_request:                # compared against the base

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      checks: write          # write the Check Run (public repos)
      pull-requests: write   # write the sticky comment (public repos)
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: rperf-dev/prperf-action@v1
        with:
          benchmark: boot
          run: bin/rails runner ""
```

The single workflow's push records the base, so that alone gives you **boot
alloc/GC compared on every PR**.

> Make sure rperf 0.11.1 or newer is in your Gemfile.

## Measure one request (the full version)

This measures the allocations and GC along an endpoint's request-handling path.
Paste three files.

### A measurement environment `config/environments/benchmark.rb`

A production-like but CI-friendly dedicated environment.

```ruby
# config/environments/benchmark.rb
require_relative "production"

Rails.application.configure do
  config.eager_load = true          # load all code, like production
  config.force_ssl = false          # avoid an SSL redirect measuring nothing
  config.hosts.clear                # drop host restrictions (for the benchmark)
  config.require_master_key = false # boot without the master key
  config.log_level = :warn
  config.consider_all_requests_local = false
end
```

### The benchmark `bench/request.rb`

```ruby
# bench/request.rb — one request through the full stack, N times
require_relative "../config/environment"
require "rack/mock"

PATH = ENV.fetch("BENCH_PATH", "/api/health")  # ← change to the endpoint you care about

app = Rails.application
build_env = -> { Rack::MockRequest.env_for(PATH, "HTTP_HOST" => "localhost") }

consume = lambda do |result|
  body = result[2]
  body.each { |_| }                 # consume the body so rendering is measured
  body.close if body.respond_to?(:close)
end

# warm up (autoload, template compilation, connection setup)
3.times { consume.call(app.call(build_env.call)) }

1_000.times { consume.call(app.call(build_env.call)) }
```

### The workflow `.github/workflows/prperf.yml`

```yaml
name: prperf
on:
  push:
    branches: [main, master]   # records the base (default branch)
  pull_request:                # compared against the base

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      checks: write          # write the Check Run (public repos)
      pull-requests: write   # write the sticky comment (public repos)
    services:
      postgres:                      # drop services and db:prepare if you don't use a DB
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports: [ "5432:5432" ]
        options: >-
          --health-cmd pg_isready --health-interval 10s
          --health-timeout 5s --health-retries 5
    env:
      RAILS_ENV: benchmark
      SECRET_KEY_BASE: dummy-for-benchmark
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/app_benchmark
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bin/rails db:prepare db:seed   # only if the request hits the DB; seed fixed data
      - uses: rperf-dev/prperf-action@v1
        with:
          benchmark: boot
          run: bin/rails runner ""
      - uses: rperf-dev/prperf-action@v1
        with:
          benchmark: request
          run: ruby bench/request.rb
```

The single workflow records the base on push to the default branch.

## The only things you change

- **`PATH`** (`bench/request.rb`) — the endpoint to measure. A **JSON/API
  endpoint is easiest** (no asset precompilation, less likely to be auth-gated).
- **`db:seed`** — if the request reads the DB, provide fixed seed data; if not,
  delete the postgres service and the db line.
- **The count (1,000)** — tune so the whole run is a few hundred ms to a few
  seconds.

## When it doesn't work

- **Empty results / all redirects** — `force_ssl` is issuing 301s, or auth is
  blocking you. The `benchmark` environment already sets `force_ssl = false`. For
  an auth-gated path, pick a public endpoint or build a signed-in env in
  `bench/request.rb`.
- **Asset-related errors** — an asset helper in the view. The quick fix is to
  measure an **API/JSON endpoint**.
- **Numbers jiggle every run** — check that the seed is fixed and the request
  has no time/randomness (see the "Writing a benchmark" checklist). Locally, run
  `RAILS_ENV=benchmark bundle exec rperf stat -- ruby bench/request.rb` twice and
  confirm alloc/GC match.
