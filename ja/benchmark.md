# ベンチマークの書き方

prperf の価値は **何を測るか** でほぼ決まります。ここが一番大事で、一番手間の
かかるところです。良いベンチマークの書き方を、具体例つきで説明します。

## ベンチマークとは(prperf の文脈で)

`run:` に渡すコマンドが「ベンチマーク」です。多くの場合は **小さな Ruby
スクリプト**(例 `bench/main.rb`)を `rperf record` で包んだもの:

```yaml
run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

action はこれを `count`(既定 3)回まわし、サーバーが中央値で base と比較します。
あなたが書くのは `bench/main.rb` の中身、つまり「**代表的な処理を一定量こなす
スクリプト**」です。

## 大原則: 「気にしている処理を、決定的に、一定量」

良いベンチには 3 条件あります。

1. **気にしている処理を通す** — そのコードに触らない PR は数字が動きません。
2. **決定的**(毎回まったく同じ仕事をする) — でないと alloc/GC がブレて、
   回帰ではないのに警告が出ます。
3. **一定量こなす** — 一瞬で終わるとサンプルが少なく不安定。

prperf の主役指標(アロケーション数・GC 回数)は決定的なので、**ベンチさえ
決定的なら、これらは PR 間で 1 個単位で安定します**。逆にベンチがブレるとこの
強みが台無しになります。

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

ポイント:

- **入力は固定**。`rand`・`Time.now`・DB・外部 API・ファイル列挙順などに
  依存しない。どうしても乱数が要るなら `srand(42)` で固定する。
- **ウォームアップ**で「初回だけ走る処理(autoload、定数初期化、JIT 的な
  ウォームアップ)」を計測対象から外す。
- **回数(ここでは 5,000)** は、全体が数百 ms〜数秒になるくらいに調整。短すぎ
  ると不安定、長すぎると CI が遅くなる。

## 決定的にするチェックリスト

- [ ] `rand` / `SecureRandom` を使っていない(or `srand` で固定)
- [ ] `Time.now` / `Date.today` に結果が依存しない
- [ ] ネットワーク・外部サービスを叩いていない
- [ ] 本物の DB ではなく固定のインメモリ/フィクスチャを使う
- [ ] ファイルの列挙順(`Dir.glob` の順序など)に依存しない
- [ ] 入力サイズが毎回同じ

## ローカルで「ブレないか」を確認する

CI に入れる前に、手元で 2〜3 回流して**アロケーション数と GC 回数が毎回同じ**
ことを確認してください。`rperf stat` が summary を stderr に出します。

```sh
bundle exec rperf stat -- ruby bench/main.rb
bundle exec rperf stat -- ruby bench/main.rb
```

2 回の `allocated_objects` と GC 回数が一致すれば決定的。バラつくなら、上の
チェックリストで非決定要素を探します。フレームグラフで中身を見たいときは:

```sh
bundle exec rperf record -o out.json.gz -- ruby bench/main.rb
bundle exec rperf report out.json.gz       # viewer が開く
```

## 何を測るか(プロジェクト別の例)

### gem / ライブラリ

公開 API を、代表的な固定入力で N 回呼ぶ。

```ruby
require "your_lib"
doc = File.read("bench/fixtures/sample.xml")   # リポジトリに固定で置く
2_000.times { YourLib.parse(doc) }
```

### Rails アプリ

- **起動(boot)** — ゼロ設定で始められる。`bin/rails runner ""` を測るだけで、
  eager load や gem 追加による起動劣化を捕まえられる(登録編参照)。
- **1 リクエスト** — テスト環境を起動し、固定のリクエストを `Rack` 経由で
  通す。固定の seed データに対して同じエンドポイントを N 回。
- **典型クエリ / サービス** — 固定のインメモリデータに対するロジックを N 回。
- **ジョブ** — `SomeJob.new.perform(fixed_args)` を N 回。

### CLI ツール

代表的なサブコマンドを固定入力で 1 回(または数回)。

## 1 つのベンチに詰め込みすぎない

巨大な「全部入り」ベンチは、どこが回帰したか分かりにくくなります。**関心ごとに
分ける**のがおすすめ。prperf は 1 コミットで**複数ベンチを独立比較**できるので、
ステップを分けて `benchmark:` 名を変えるだけです(登録編「複数ベンチマーク」)。

```yaml
- uses: rperf-dev/prperf-action@v1
  with: { benchmark: parse,     run: 'bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/parse.rb' }
- uses: rperf-dev/prperf-action@v1
  with: { benchmark: serialize, run: 'bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/serialize.rb' }
```

## アンチパターン

- **テストスイートをそのまま測る**(`rperf record -- rspec`)。PR でテストを
  足すだけで alloc が増え、回帰と区別できない。やるなら正規化が要る。
- **乱数・時刻・ネットワーク依存** → 毎回ブレて誤検知。
- **短すぎる** → 時間がブレ、alloc も小さすぎて差が見えない。
- **関心の薄い経路を測る** → PR が触らず、毎回「変化なし」。
- **本物の外部依存(API・DB)** → ネットワーク次第でブレる。

## 閾値とのつながり

ベンチが決定的だと、**きつい相対閾値**(例 `alloc: "+5%"`)を誤検知なしで
かけられます。ブレるベンチだと閾値を緩めるしかなく、信号が弱くなります。
つまり「良いベンチ = 鋭い閾値」。まずは 1 本、自分のいちばん大事な経路を
決定的に測るところから始めてください。
