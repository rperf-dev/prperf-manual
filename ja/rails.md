# Rails クイックスタート

「何を測るか」で悩む前に、**ほぼコピペで動く** Rails 向けの出発点を 2 段階で
示します。まず ① で「数字が出る」体験を 30 秒で作り、必要なら ② に進んでください。

## ① まず boot を測る(ファイル追加ゼロ)

`bin/rails runner ""` は **アプリを起動して何もせず終わる**ので、起動そのものを
計測できます。gem 追加や initializer の重さ、autoload 構成の変化を捕まえられ、
**決定的**で、追加ファイルも DB も要りません。

`.github/workflows/prperf.yml`(PR 用)をそのまま貼ってください:

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
          benchmark: boot
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- bin/rails runner ""
```

base 用に、同じ steps を `on: push` / `branches: [main]` にした
`.github/workflows/prperf-base.yml` も置きます(登録編参照)。これだけで
**boot の alloc/GC が PR ごとに比較**されます。

> rperf 0.10 以上を Gemfile に入れておいてください。

## ② 1 リクエストを測る(本格版)

実アプリの「1 リクエストの重さ」を測ります。3 ファイルを貼るだけです。

### (a) 計測用の環境 `config/environments/benchmark.rb`

production 相当だが CI で動かしやすい専用環境を作ります。

```ruby
# config/environments/benchmark.rb
require_relative "production"

Rails.application.configure do
  config.eager_load = true          # 本番同様に全コードを読む
  config.force_ssl = false          # SSL リダイレクトで計測が空になるのを防ぐ
  config.hosts.clear                # ホスト制限を外す(ベンチ用)
  config.require_master_key = false # master key 無しでも起動
  config.log_level = :warn
  config.consider_all_requests_local = false
end
```

### (b) ベンチ本体 `bench/request.rb`

```ruby
# bench/request.rb — フルスタックを通る 1 リクエストを N 回
require_relative "../config/environment"
require "rack/mock"

PATH = ENV.fetch("BENCH_PATH", "/api/health")  # ← 測りたいエンドポイントに変更

app = Rails.application
build_env = -> { Rack::MockRequest.env_for(PATH, "HTTP_HOST" => "localhost") }

consume = lambda do |result|
  body = result[2]
  body.each { |_| }                 # body を消費してレンダリングまで測る
  body.close if body.respond_to?(:close)
end

# ウォームアップ(autoload・テンプレートコンパイル・コネクション確立)
3.times { consume.call(app.call(build_env.call)) }

1_000.times { consume.call(app.call(build_env.call)) }
```

### (c) ワークフロー `.github/workflows/prperf.yml`

```yaml
name: prperf
on: pull_request

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    services:
      postgres:                      # DB を使わないなら services と db:prepare は消す
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
      PRPERF_DEFAULT_THRESHOLDS: |
        alloc: "+10%"
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bin/rails db:prepare db:seed   # DB を使う場合のみ。seed は固定データで
      - uses: rperf-dev/prperf-action@v1
        with:
          benchmark: boot
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- bin/rails runner ""
      - uses: rperf-dev/prperf-action@v1
        with:
          benchmark: request
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/request.rb
```

base 用も同じ steps で `on: push` 版を作ります。

## 変えるのはここだけ

- **`PATH`**(`bench/request.rb`)— 測りたいエンドポイント。**JSON/API
  エンドポイントが楽**(アセットのプリコンパイル不要、認証で弾かれにくい)。
- **`db:seed`** — リクエストが DB を引くなら、固定の seed データを用意。
  引かないなら postgres service と db 行ごと削除。
- **回数(1,000)** — 全体が数百 ms〜数秒になるよう調整。

## うまくいかないとき

- **空の結果/リダイレクトばかり** — `force_ssl` で 301、または認証で弾かれて
  いる。`benchmark` 環境で `force_ssl=false` 済み。認証が要る経路なら、公開
  エンドポイントを選ぶか、`bench/request.rb` でログイン済み env を組む。
- **アセット関連のエラー** — ビュー内の asset ヘルパーが原因。**API/JSON
  エンドポイント**を選ぶのが手っ取り早い。
- **数字が毎回ブレる** — seed が固定か、リクエストに時刻/乱数が混ざっていないか
  を確認(「ベンチマークの書き方」のチェックリスト)。ローカルで
  `RAILS_ENV=benchmark bundle exec rperf stat -- ruby bench/request.rb` を 2 回
  流して alloc/GC が一致するか見てください。
