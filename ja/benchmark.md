# ベンチマークの書き方

prperf が出す数字は、何を測るかでほぼ決まります。
ベンチマークの設計は、この章で最も結果を左右し、最も手間がかかるところです。

## ベンチマークとは

prperf にとってのベンチマークは、`run:` に渡すコマンドです。
多くの場合は小さな Ruby スクリプト（例 `bench/main.rb`）を走らせるだけです。

```yaml
run: ruby bench/main.rb
```

`run:` には計測したいコマンドだけを書き、Action が `rperf record` で自動的に包みます。
rperf は Gemfile に入れておきます（0.10 以上。Action が `bundle exec rperf` で呼ぶためです）。
action はこれを `count` 回（既定 3）まわし、サーバーが中央値を base と比較します。
あなたが書くのは `bench/main.rb` の中身、つまり代表的な処理を一定量こなすスクリプトです。

## 良いベンチマークの条件

良いベンチマークは次の三つを満たします。

1. **気にしている処理を通す**。そのコードに触らない PR では数字が動きません。
2. **決定的である**（毎回まったく同じ仕事をする）。さもないと alloc や GC がブレて、回帰ではないのに警告が出ます。
3. **一定量こなす**。一瞬で終わるとサンプルが少なく、結果が不安定になります。

回帰判定の軸になるのはアロケーション数です。
ベンチマークが決定的なら、アロケーション数は PR 間で 1 個単位まで安定し、わずかな増加も捉えられます。
GC 回数も決定的で数えやすいので、併せて表示します。
逆にベンチマークがブレると、この安定性が失われます。

## 書き方

`bench/main.rb` は、固定入力を一度用意し、ウォームアップしてから、本番のループを十分な回数くりかえす形が基本です。

```ruby
# bench/main.rb
require "json"
require_relative "../config/environment"   # 必要なら（Rails 等）

# 1) 固定の入力を一度だけ用意（乱数・時刻・ネットワークを使わない）
DATA = { "users" => Array.new(100) { |i| { "id" => i, "name" => "user#{i}" } } }

# 2) ウォームアップ（初回限りの遅延読み込み・初期化を計測から外す）
JSON.generate(DATA)

# 3) 本番: 十分な回数くりかえす
5_000.times do
  JSON.generate(DATA)
end
```

入力は固定し、`rand` や `Time.now`、DB、外部 API、ファイルの列挙順などに依存させません。
どうしても乱数が要るなら `srand(42)` で固定します。
ウォームアップで、初回だけ走る処理（autoload、定数初期化、遅延ロード）を計測対象から外します。
回数（ここでは 5,000）は、全体が数百 ms から数秒になるくらいに調整します。
短すぎると不安定になり、長すぎると CI が遅くなります。

## 決定的か確かめる

CI に入れる前に、手元で 2〜3 回流して、アロケーション数と GC 回数が毎回同じことを確認します。
`rperf stat` が summary を stderr に出します。

```sh
bundle exec rperf stat -- ruby bench/main.rb
bundle exec rperf stat -- ruby bench/main.rb
```

2 回とも `allocated_objects` と GC 回数が一致すれば、そのベンチマークは決定的です。
ブレるなら、次を一つずつ潰します。

- [ ] `rand` や `SecureRandom` を使っていない（使うなら `srand` で固定）
- [ ] `Time.now` や `Date.today` に結果が依存しない
- [ ] ネットワークや外部サービスを叩いていない
- [ ] 変動する DB 状態に依存しない（固定のインメモリデータ、fixture、固定 seed を使う）
- [ ] ファイルの列挙順（`Dir.glob` の順序など）に依存しない
- [ ] 入力サイズが毎回同じ

原因の見当をつけるには、フレームグラフで中身を見ます。

```sh
bundle exec rperf record -o out.json.gz -- ruby bench/main.rb
bundle exec rperf report out.json.gz       # viewer が開く
```

非決定要素を固定したら、もう一度 `rperf stat` で一致を確認してから CI に入れます。

## 前準備（任意）

ベンチマークの前に1回だけ走らせたい準備があれば、`prepare_run:` に書きます。
fixture の生成、DB の seed、アセットのビルドなどが該当します。
計測の前に1回だけ実行され、計測には含まれません。

```yaml
- uses: rperf-dev/prperf-action@v1
  with:
    prepare_run: bin/rails db:prepare db:seed   # 計測前に1回だけ
    run: ruby bench/request.rb
```

失敗するとステップが落ちます。
毎回同じ状態になるよう、固定の seed や入力を使ってください。

## プロジェクト別の例

ワークフローは「登録編」のものをそのまま使い、`run:` を各 `bench/*.rb` に向けるだけです。
前準備が要るベンチ（fixture の生成、DB の seed、アセットのビルドなど）だけ、`prepare_run:` を足します。

### gem / ライブラリ

公開 API を、決定的な固定入力で N 回呼びます。
入力と呼び出し回数を固定したまま、公開 API の回帰だけを測れます。

```ruby
# bench/main.rb
require "your_gem"

# 固定入力を決定的に組む（乱数・時刻・ネットワークを使わない）
DATA = { "items" => Array.new(200) { |i| { "id" => i, "name" => "item-#{i}" } } }

YourGem.encode(DATA)                 # ウォームアップ
5_000.times { YourGem.encode(DATA) }
```

変えるのは `require`、固定入力の `DATA`、呼ぶ API、回数です。
既存の `benchmark/` スクリプト（benchmark-ips など）も流用できますが、時間ベースのループは反復回数が変わって alloc がブレるので、回数固定のループにしてください。

### Sinatra / Rack アプリ

Rack アプリなら何でも、1 リクエストをフルスタックで N 回通します。

```ruby
# bench/request.rb
require_relative "../app"            # あなたの Sinatra/Rack アプリを読み込む
require "rack/mock"

app  = Sinatra::Application           # クラシック。モジュラーなら app = MyApp
PATH = ENV.fetch("BENCH_PATH", "/")   # 測りたいパスに変更
make = -> { Rack::MockRequest.env_for(PATH, "HTTP_HOST" => "localhost") }
pump = ->(r) { b = r[2]; b.each { |_| }; b.close if b.respond_to?(:close) }

3.times    { pump.call(app.call(make.call)) }   # ウォームアップ
2_000.times { pump.call(app.call(make.call)) }
```

`run:` を `ruby bench/request.rb` にします。
変えるのは、アプリの読み込み行、`app`（`config.ru` で `run` しているオブジェクト）、`PATH`、回数です。
DB を引くなら、前準備に seed を、ワークフローに postgres サービスを足します（「Rails クイックスタート」を参照）。

### CLI / 素の Ruby

エントリポイントをプロセス内で固定引数で N 回呼ぶと、起動コストや外部状態の影響を避けやすく、サンプルも安定します。

```ruby
# bench/main.rb
require_relative "../lib/my_cli"

ARGS = %w[build --format json]        # 固定の引数
200.times { MyCli.run(ARGS) }         # あなたのエントリポイントを呼ぶ
```

実行ファイルそのものを測るなら、十分な仕事量があることを確認してから `run:` に直接渡します。
1 回の起動が短いとサンプルが少なく不安定なので、ループで回すか、上のプロセス内ループにしてください。

### Rails アプリ

Rails は次章「Rails クイックスタート」で扱います。
boot、エンドポイント、典型的なクエリ、ジョブの測り方はそこにまとめています。
Roda や grape は Rack アプリなので「Sinatra / Rack」と同じ、Hanami は Rails と同じ発想で測れます。

## やってはいけないこと

次のような測り方は、PR の変更内容と無関係な要因で数字が動くので避けます。

- **テストスイートをそのまま測る**（`rperf record -- rspec`）。PR でテストを足すだけで alloc が増え、回帰と区別できません。
- **乱数や時刻、ネットワークに依存する**。毎回ブレて誤検知になります。
- **短すぎる**。時間がブレ、alloc も小さすぎて差が見えません。
- **関心の薄い経路を測る**。PR が触らず、毎回「変化なし」になります。
- **本物の外部依存（API や DB）を使う**。ネットワーク次第でブレます。

巨大な全部入りベンチマークも、どこが回帰したか分かりにくくなるので避けます。
関心ごとに分け、1 コミットで複数のベンチマークとして測ると、回帰した処理を Check Run と diff で追いやすくなります（分け方は「登録編」の複数ベンチマークを参照）。

## 閾値とのつながり

ベンチマークが決定的だと、きつい相対閾値（例 `alloc: "+5%"`）を誤検知なしでかけられます。
ブレるベンチマークだと閾値を緩めるしかなく、信号が弱くなります。
まずは 1 本、PR が最も触りやすい経路を決定的に測り、base と head の差が出る状態を作るところから始めてください。
