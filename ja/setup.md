# 登録編

prperf を使い始めるまでの流れです。
所要 10〜15 分。
サービスの概要は「このサービスとは」を参照してください。

やることは 3 つです。

1. GitHub App をインストールする
2. ベンチマークを用意する
3. それを実行するワークフローを追加する

## 前提

rperf 0.10 以上を Gemfile に入れていることが必要です。
prperf はプロファイルに埋め込まれた `meta` と `summary` を使います。
古い rperf では action が明確なエラーで止まります。

計測対象のベンチマークコマンドがあることも必要です（後述）。

対象は public リポジトリです。
private リポジトリは有料プランで提供します（現在は無料βのため public 専用）。

## GitHub App をインストール

prperf の GitHub App ページからリポジトリにインストールします。
これにより、prperf がそのリポジトリの Check Run と PR コメントを書けるようになります。

## ベンチマークを用意する

何を測るかが、検知できる回帰の範囲を決めます。
良いベンチマークは、決定的で、気にしている経路を通り、そこそこの規模があるものです。
Rails プロジェクトなら、とりあえず起動（boot）の計測から試すのが手軽です。
書き方やほかのプロジェクトの例は「ベンチマークの書き方」を参照してください。

## ワークフローを追加

用意したベンチを実行するワークフローを追加します。
`push`（既定ブランチ）と `pull_request` の両方をトリガにします。
push が base を、pull_request が head を供給します。
base と head の区別は、prperf が OIDC トークンの ref から判定します。

既定ブランチは `main` と `master` の両方を書いておけば、どちらでも動きます。
既定ブランチへの push が一度も無いと比較対象が無く、「比較対象なし、今回の数値のみ」になります。

```yaml
# .github/workflows/prperf.yml
name: prperf
on:
  push:
    branches: [ main, master ]   # base を記録（既定ブランチ）
  pull_request:                  # head を base と比較

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read         # checkout 用
      id-token: write        # OIDC アップロードに必須
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

`run:` には、rperf に profile を 1 本以上書かせるコマンドを渡します。
出力先は `--snapshot-dir "$PRPERF_DIR"` を指定します（`$PRPERF_DIR` は action が用意します）。

`permissions: id-token: write` は必ず付けてください。
これが無いと OIDC トークンが取れず、アップロードできません。
`contents: read` は `actions/checkout` がリポジトリを取得するためです。
`permissions:` を書くと、挙げていない権限は none になるので、両方を明示します。

## 閾値とコメントの設定（任意）

閾値は、base から head への各指標の増加（アロケーション、GC、時間など）に上限を設けます。
超えると Check Run に ⚠️ が付き、`comment` 設定に応じて PR コメントが出ます。
設定は任意で、設定しなければ数字は出ますが警告もコメントも付きません。
回帰したら警告してほしいときだけ設定します。

設定は全部ワークフローに書きます（別ファイルは不要）。
全体設定を job の `env` に一度だけ書き、必要ならベンチごとに上書きします。

```yaml
jobs:
  bench:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write }
    env:
      PRPERF_DEFAULT_THRESHOLDS: |     # 全ベンチ共通の既定
        alloc: "+10%"
        total_ms: "+20%"
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with: { bundler-cache: true }
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/main.rb
```

閾値のキーは次のとおりです。

| キー | 値の例 | 意味 |
|---|---|---|
| `alloc` | `"+10%"` / `"+5000"` | アロケーション数の増加(相対/絶対) |
| `gc_count` | `"+2"` | GC 回数(minor+major)の増加 |
| `total_ms` | `"+20%"` | 実行時間(ノイズが大きいので相対推奨) |
| `cpu_ms` | `"+15%"` | CPU 時間 |
| `method` | `{ "JSON.generate": "15%" }` | メソッドの self 占有率の絶対値超過 |

値は、summary 系が `"+N%"`（相対）か `"+N"`（絶対）、method 系が `"N%"` です。
不正な値は無視され、Check Run に警告が 1 行出ます（CI は落ちません）。
相対閾値（`+10%`）はベンチをまたいで素直に機能します。
絶対閾値や method はベンチごとに意味が変わるので、必要なときだけベンチ別に上書きしてください。

コメントの出し方は `comment` 入力で制御します（既定 `on_threshold`）。

| 値 | 挙動 |
|---|---|
| `on_threshold` | 閾値超過時のみ PR コメント(既定) |
| `always` | 毎回コメント |
| `never` | コメントしない(Check Run だけ) |

コメントは PR につき 1 通で、push のたびに同じ sticky コメントを編集します（通知が増えません）。

## action の入力一覧

| 入力 | 既定 | 説明 |
|---|---|---|
| `run` | (必須) | 計測コマンド。`.json.gz` を 1 本以上吐くこと |
| `prepare_run` | `""` | 計測前に1回だけ走るセットアップ（fixture 生成や seed など）。計測には含めない |
| `count` | `3` | 計測回数。サーバーは中央値で比較 |
| `benchmark` | `default` | ベンチ系列名。1 コミットで複数ベンチを独立比較できる |
| `thresholds` | `""` | このベンチの閾値(全体設定をキー単位で上書き) |
| `comment` | `on_threshold` | コメントの出し方 |
| `server` | `https://prperf.atdot.net` | prperf サーバー(差し替え可) |
| `upload` | `true` | `false` で計測のみ(アップロードしない) |

通常は `run` だけを指定し、必要に応じて `benchmark`、`thresholds`、`comment` を足します。

## 複数ベンチマーク

1 コミットを複数のベンチで測れます。
ステップを分けて、それぞれに違う `benchmark` 名を付けるだけです。
サーバーは各ベンチを自分の base と比較し、1 つの Check Run にまとめて表示します。

```yaml
- uses: rperf-dev/prperf-action@v1
  with:
    benchmark: boot
    run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- bin/rails runner ""
- uses: rperf-dev/prperf-action@v1
  with:
    benchmark: render
    run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- ruby bench/render.rb
```

ワークフローが 1 本なので、base 側（push）と PR 側で同じ `benchmark` 名が自然に揃います。
各系列が自分の base を持つには、この一致が必要です。

## 動作確認

1. まず既定ブランチ（main または master）に push します。ワークフローが走り、base スナップショットがサーバーに届きます。
2. 適当な PR を作る。ワークフローが走り、Check Run に数字が出れば成功です。
3. アップロード結果のリンクは各ジョブの Summary にも出ます。

Check Run に数字が出れば、base の記録と PR 側の比較がどちらも動いています。

## 制約

fork からの PR は計測できません。
GitHub は fork 起因のワークフローに `id-token: write` を与えないため、OIDC トークンが取れません。
同一リポジトリのブランチ PR は問題ありません。

アップロードの失敗（プラン上限、レート制限、サーバーエラー）は警告のみで、ステップは成功扱いです。
計測コマンド自体の失敗だけがステップを落とします。

無料β期間中は public リポジトリのみです。
private は有料プランで近日提供予定です。
