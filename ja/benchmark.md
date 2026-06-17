# ベンチマークの書き方

prperf が出す数字は、何を測るかでほぼ決まります。
ベンチマークの設計は、この章で扱う作業のなかで最も結果を左右し、最も手間がかかります。
良いベンチマークの書き方を、具体例とともに説明します。

## prperf におけるベンチマーク

`run:` に渡すコマンドがベンチマークです。
多くの場合は小さな Ruby スクリプト(例 `bench/main.rb`)を `rperf record` で包みます。

```yaml
run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

rperf は Gemfile に入れておきます（0.10 以上。`bundle exec rperf` で呼ぶためです）。
action はこれを `count`(既定 3)回まわし、サーバーが中央値で base と比較します。
あなたが書くのは `bench/main.rb` の中身、つまり代表的な処理を一定量こなすスクリプトです。

## 良いベンチマークの三条件

良いベンチマークは次の三つを満たします。

1. **気にしている処理を通す**。そのコードに触らない PR では数字が動きません。
2. **決定的である**(毎回まったく同じ仕事をする)。さもないと alloc や GC がブレて、回帰ではないのに警告が出ます。
3. **一定量こなす**。一瞬で終わるとサンプルが少なく、結果が不安定になります。

回帰判定の軸になるのはアロケーション数です。
ベンチマークが決定的なら、アロケーション数は PR 間で 1 個単位まで安定し、わずかな増加も捉えられます。
GC 回数も決定的で数えやすいので、併せて表示します。
逆にベンチマークがブレると、この安定性が失われます。

## 骨格(テンプレート)

```ruby
# bench/main.rb
require "json"
require_relative "../config/environment"   # 必要なら(Rails 等)

# 1) 固定の入力を一度だけ用意(乱数・時刻・ネットワークを使わない)
DATA = { "users" => Array.new(100) { |i| { "id" => i, "name" => "user#{i}" } } }

# 2) ウォームアップ(初回限りの遅延読み込み・初期化を計測から外す)
JSON.generate(DATA)

# 3) 本番: 十分な回数くりかえす
5_000.times do
  JSON.generate(DATA)
end
```

入力は固定します。
`rand`、`Time.now`、DB、外部 API、ファイル列挙順などに依存させません。
どうしても乱数が要るなら `srand(42)` で固定します。

ウォームアップで、初回だけ走る処理(autoload、定数初期化、JIT 的なウォームアップ)を計測対象から外します。

回数(ここでは 5,000)は、全体が数百 ms から数秒になるくらいに調整します。
短すぎると不安定になり、長すぎると CI が遅くなります。

## 決定的にするチェックリスト

- [ ] `rand` / `SecureRandom` を使っていない(or `srand` で固定)
- [ ] `Time.now` / `Date.today` に結果が依存しない
- [ ] ネットワーク・外部サービスを叩いていない
- [ ] 変動する DB 状態に依存しない(固定のインメモリデータ、fixture、固定 seed を使う)
- [ ] ファイルの列挙順(`Dir.glob` の順序など)に依存しない
- [ ] 入力サイズが毎回同じ

ひとつでも当てはまらなければ、固定入力や seed に置き換えてから CI に入れます。

## ローカルでブレないことを確認する

CI に入れる前に、手元で 2 回から 3 回流して、アロケーション数と GC 回数が毎回同じことを確認してください。
`rperf stat` が summary を stderr に出します。

```sh
bundle exec rperf stat -- ruby bench/main.rb
bundle exec rperf stat -- ruby bench/main.rb
```

2 回とも `allocated_objects` と GC 回数が一致すれば、そのベンチマークは決定的です。
バラつくなら、上のチェックリストで非決定要素を探します。
フレームグラフで中身を見たいときは次のようにします。

```sh
bundle exec rperf record -o out.json.gz -- ruby bench/main.rb
bundle exec rperf report out.json.gz       # viewer が開く
```

## 前準備（任意）

ベンチマークの前に1回だけ走らせたい準備があれば、`prepare_run:` に書きます。
fixture の生成、DB の seed、アセットのビルドなどが該当します。
計測の前に1回だけ実行され、計測には含まれません。

```yaml
- uses: rperf-dev/prperf-action@v1
  with:
    prepare_run: bin/rails db:prepare db:seed   # 計測前に1回だけ
    run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/request.rb
```

失敗するとステップが落ちます。
毎回同じ状態になるよう、固定の seed や入力を使ってください。

## プロジェクト別の例

プロジェクトごとに、ほぼコピペで始められる例を示します。
gem やライブラリ、Sinatra や Rack アプリ、CLI や素の Ruby は、どれも `bench/*.rb` を 1 つ置いて `run:` をそれに向けるだけで、DB も専用環境も要りません。
まず共通の最小ワークフローを示し、続いてプロジェクトごとに `bench/*.rb` の中身を説明します。

非 Rails はだいたいこの最小ワークフローで足ります。
差し替えるのは `run:` だけです。
ワークフローは 1 本で、push と pull_request の両方をトリガにします。
push で記録した計測が base のスナップショットになり、PR の head がその比較対象になります。
prperf は OIDC の ref で base と head を判別するので、ファイルを分ける必要はありません。

`.github/workflows/prperf.yml`:

```yaml
name: prperf
on:
  push:
    branches: [main, master]
  pull_request:

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

rperf 0.10 以上を Gemfile に入れてください。

### gem / ライブラリ

公開 API を、決定的な固定入力で N 回呼びます。
この形なら、入力と呼び出し回数を固定したまま公開 API の回帰だけを測れます。

```ruby
# bench/main.rb
require "your_gem"

# 固定入力を決定的に組む(乱数・時刻・ネットワークを使わない)
DATA = { "items" => Array.new(200) { |i| { "id" => i, "name" => "item-#{i}" } } }

YourGem.encode(DATA)                 # ウォームアップ
5_000.times { YourGem.encode(DATA) }
```

変えるのは `require`、固定入力の `DATA`、呼ぶ API、回数です。

> 既存の `benchmark/` スクリプト(benchmark-ips など)があっても流用できますが、prperf では**回数固定のループ**にしてください。
> 時間ベースのループだと反復回数が変わって alloc がブレます。

### Sinatra / Rack アプリ

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

ワークフローの `run:` を `ruby bench/request.rb` にします。
変えるのは、アプリの読み込み行、`app`(`config.ru` で `run` しているオブジェクトを指す)、`PATH`、回数です。
DB を引くなら共通ワークフローに postgres service と seed を足します(Rails クイックスタート②を参照)。

### CLI / 素の Ruby

エントリポイントを**プロセス内で**固定引数で N 回呼ぶと、起動コストや外部状態の影響を避けやすくなります。
反復のたびに同じ仕事を繰り返すので、サンプルが安定します。

```ruby
# bench/main.rb
require_relative "../lib/my_cli"

ARGS = %w[build --format json]        # 固定の引数
200.times { MyCli.run(ARGS) }         # あなたのエントリポイントを呼ぶ
```

実行ファイルそのものを測りたいなら、十分な仕事量があることを確認した上で次のようにします。

```yaml
run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby exe/mycli build fixtures/sample.txt
```

ただし 1 回の起動が短いとサンプルが少なく不安定です。
ループで回す(中で大きい固定入力を処理する)か、上のプロセス内ループにしてください。

### その他のフレームワーク

Roda や grape は Rack アプリなので、上の「Sinatra / Rack」と同じやり方で測れます。
Hanami は Rails と同じ発想で、boot はアプリのブートを、リクエストは Rack 経由で測れます。
迷ったら、まずは一番大事な経路を 1 本、決定的に測ってください。

### Rails アプリ

Rails は専用の章があるので、ここでは詳細を書きません。
boot やエンドポイント、典型クエリ、ジョブの測り方は「Rails クイックスタート」を参照してください。

## 1 つのベンチマークに詰め込みすぎない

巨大な全部入りベンチマークは、どこが回帰したか分かりにくくなります。
関心ごとに分けると、回帰した処理を Check Run と diff で追いやすくなります。
prperf は 1 コミットで複数ベンチマークを独立に比較できるので、ステップを分けて `benchmark:` 名を変えるだけです(登録編「複数ベンチマーク」)。

```yaml
- uses: rperf-dev/prperf-action@v1
  with: { benchmark: parse,     run: 'bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/parse.rb' }
- uses: rperf-dev/prperf-action@v1
  with: { benchmark: serialize, run: 'bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/serialize.rb' }
```

## アンチパターン

次のような測り方は避けます。

- **テストスイートをそのまま測る**(`rperf record -- rspec`)。PR でテストを足すだけで alloc が増え、回帰と区別できません。やるなら正規化が要ります。
- **乱数・時刻・ネットワーク依存**。毎回ブレて誤検知になります。
- **短すぎる**。時間がブレ、alloc も小さすぎて差が見えません。
- **関心の薄い経路を測る**。PR が触らず、毎回「変化なし」になります。
- **本物の外部依存(API・DB)**。ネットワーク次第でブレます。

どれも、PR の変更内容と無関係な要因で数字が動くのが共通点です。

## 閾値とのつながり

ベンチマークが決定的だと、きつい相対閾値(例 `alloc: "+5%"`)を誤検知なしでかけられます。
ブレるベンチマークだと閾値を緩めるしかなく、信号が弱くなります。
良いベンチマークは鋭い閾値を支えます。
まずは 1 本、PR が最も触りやすいコードを通る経路を決定的に測り、base と head の差が出る状態を作るところから始めてください。
