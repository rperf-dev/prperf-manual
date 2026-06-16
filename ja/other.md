# 他のプロジェクトのクイックスタート

Rails 以外もほぼコピペで始められます。**gem / ライブラリ**、**Sinatra / Rack
アプリ**、**CLI / 素の Ruby** の 3 つを示します。どれも `bench/*.rb` を 1 つ
置いて、`run:` をそれに向けるだけ。DB も専用環境も要りません。

## 共通のワークフロー

非 Rails はだいたいこの最小ワークフローで足ります(`run:` だけ差し替える)。
PR 用 `.github/workflows/prperf.yml`:

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

base 用に、同じ steps で `on: push` / `branches: [main]` 版も置きます
(登録編参照)。rperf 0.10 以上を Gemfile に。

## gem / ライブラリ

公開 API を、決定的な固定入力で N 回呼びます。

```ruby
# bench/main.rb
require "your_gem"

# 固定入力を決定的に組む(乱数・時刻・ネットワークを使わない)
DATA = { "items" => Array.new(200) { |i| { "id" => i, "name" => "item-#{i}" } } }

YourGem.encode(DATA)                 # ウォームアップ
5_000.times { YourGem.encode(DATA) }
```

**変えるのはここ**: `require`、`DATA`(固定入力)、呼ぶ API、回数。

> 既存の `benchmark/` スクリプト(benchmark-ips など)があっても流用できますが、
> prperf では **回数固定のループ**にしてください。時間ベースのループだと反復回数が
> 変わって alloc がブレます。

## Sinatra / Rack アプリ

Rack アプリなら何でも、1 リクエストをフルスタックで N 回通します。

```ruby
# bench/request.rb
require_relative "../app"            # あなたの Sinatra/Rack アプリを読み込む
require "rack/mock"

app  = Sinatra::Application           # クラシック。モジュラーなら app = MyApp
PATH = ENV.fetch("BENCH_PATH", "/")   # ← 測りたいパスに変更
make = -> { Rack::MockRequest.env_for(PATH, "HTTP_HOST" => "localhost") }
pump = ->(r) { b = r[2]; b.each { |_| }; b.close if b.respond_to?(:close) }

3.times    { pump.call(app.call(make.call)) }   # ウォームアップ
2_000.times { pump.call(app.call(make.call)) }
```

ワークフローの `run:` を `ruby bench/request.rb` に。

**変えるのはここ**: アプリの読み込み行と `app`(`config.ru` で `run` している
オブジェクトを指す)、`PATH`、回数。DB を引くなら共通ワークフローに postgres
service と seed を足します(Rails クイックスタート②を参照)。

## CLI / 素の Ruby

エントリポイントを**プロセス内で**固定引数で N 回呼ぶのが、サンプルが安定して
おすすめです。

```ruby
# bench/main.rb
require_relative "../lib/my_cli"

ARGS = %w[build --format json]        # 固定の引数
200.times { MyCli.run(ARGS) }         # あなたのエントリポイントを呼ぶ
```

実行ファイルそのものを測りたいなら、十分な仕事量があることを確認した上で:

```yaml
run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby exe/mycli build fixtures/sample.txt
```

ただし 1 回の起動が短いとサンプルが少なく不安定です。**ループで回す**(中で
大きい固定入力を処理する)か、上のプロセス内ループにしてください。

## その他のフレームワーク

- **Hanami / Roda / grape など** — Roda・grape は Rack アプリなので上の
  「Sinatra / Rack」と同じやり方。Hanami は Rails と同じ発想で、boot は
  アプリのブートを、リクエストは Rack 経由で測れます。

迷ったら、まずは一番大事な経路を 1 本、決定的に。詳しくは「ベンチマークの
書き方」を参照してください。
