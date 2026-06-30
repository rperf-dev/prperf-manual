# Rails クイックスタート

「何を測るか」で悩む前に、ほぼコピペで動く Rails 向けの出発点を 2 段階で示します。

まず ① で「数字が出る」体験を 30 秒で作り、必要なら ② に進んでください。

## まず boot を測る(ファイル追加ゼロ)

`bin/rails runner ""` はアプリを起動して何もせず終わるので、起動そのものを計測できます。

gem 追加や initializer の重さ、autoload 構成の変化を捕まえられます。
結果は決定的で、追加ファイルも DB も要りません。

`.github/workflows/prperf.yml` をそのまま貼ってください。

```yaml
name: prperf
on:
  push:
    branches: [main, master]   # base を記録(既定ブランチ)
  pull_request:                # PR を base と比較

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      checks: write
      pull-requests: write
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

この 1 本で、既定ブランチ(main または master)への push が記録した boot の base と、PR の head が比較されます。

> rperf 0.11 以上を Gemfile に入れておいてください。

## 1 リクエストを測る(本格版)

1 つのエンドポイントを実際に動かし、リクエスト処理が通る経路のアロケーションと GC を測ります。

3 ファイルを貼るだけです。

### 計測用の環境 `config/environments/benchmark.rb`

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

### ベンチ本体 `bench/request.rb`

アプリを起動し、固定のエンドポイントへのリクエストを Rack 経由で N 回通すスクリプトです。
レスポンスの body まで消費して、レンダリングまで含めて測ります。
最初の数回はウォームアップとして、autoload やテンプレートのコンパイルを計測から外します。

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

### ワークフロー `.github/workflows/prperf.yml`

```yaml
name: prperf
on:
  push:
    branches: [main, master]   # base を記録(既定ブランチ)
  pull_request:                # PR を base と比較

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      checks: write
      pull-requests: write
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
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bin/rails db:prepare db:seed   # DB を使う場合のみ。seed は固定データで
      - uses: rperf-dev/prperf-action@v1
        with:
          benchmark: boot
          run: bin/rails runner ""
      - uses: rperf-dev/prperf-action@v1
        with:
          benchmark: request
          run: ruby bench/request.rb
```

この場合も、既定ブランチへの push が base を記録し、PR の head が比較されます。
boot も残すと、起動の回帰とリクエストの回帰を同じ Check Run で別系列として見られます。

## 変えるのはここだけ

- **`PATH`**(`bench/request.rb`)。測りたいエンドポイントです。JSON/API エンドポイントが楽です(アセットのプリコンパイル不要、認証で弾かれにくい)。
- **`db:seed`**。リクエストが DB を引くなら、固定の seed データを用意します。引かないなら postgres service と db 行ごと削除します。
- **回数(1,000)**。全体が数百 ms から数秒になるよう調整します。

まず `PATH` と seed を固定し、それでもブレる場合に回数や対象経路を見直します。

## うまくいかないとき

- **空の結果やリダイレクトばかり**。`force_ssl` で 301、または認証で弾かれています。`benchmark` 環境では `force_ssl=false` 済みです。認証が要る経路なら、公開エンドポイントを選ぶか、`bench/request.rb` でログイン済み env を組んでください。
- **アセット関連のエラー**。ビュー内の asset ヘルパーが原因です。API/JSON エンドポイントを選ぶと、asset ヘルパーの影響を避けやすくなります。
- **数字が毎回ブレる**。seed が固定か、リクエストに時刻や乱数が混ざっていないかを確認してください(「ベンチマークの書き方」のチェックリスト)。ローカルで `RAILS_ENV=benchmark bundle exec rperf stat -- ruby bench/request.rb` を 2 回流して alloc/GC が一致するか見てください。

いずれも、結果が決定的になるよう入力を固定してから、計測の対象や回数を調整するのが近道です。
