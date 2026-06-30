# 登録編

prperf を使い始めるまでの流れです。
所要 10〜15 分。
サービスの概要は「このサービスとは」を参照してください。

public リポジトリでやることは 2 つです。

1. ベンチマークを用意する
2. それを実行するワークフローを追加する

public リポジトリは GitHub App のインストールが要りません。
private リポジトリだけ、これに加えて prperf の GitHub App をインストールします（後述）。

## 前提

rperf 0.11.1 以降を Gemfile に入れていることが必要です（Bundler プロジェクトの場合）。
Action は `bundle exec rperf` で動かすので、Gemfile に入れておけば CLI と版が揃い、衝突しません。
Gemfile を持たないプロジェクトでは、Action が rperf を自分でインストールするので追加は要りません。

計測対象のベンチマークコマンドがあることも必要です（後述）。

互換性は rperf gem の版ではなく、プロファイルの format_version で決まります。
サーバは読める format なら版を問わず受理し、新しすぎる format は黙って誤読せず明示エラーで弾きます。

## public と private

public リポジトリは、GitHub App のインストールが要りません。
Action がワークフローの `GITHUB_TOKEN` を使って、Check Run と sticky コメントを自分で書きます。
あとはワークフローを追加するだけです。

private リポジトリは、prperf の GitHub App をインストールします。
App がサーバ側で Check Run を書きます。
無料β期間中は public も private も無料で使えます。
private は prperf の GitHub App をインストールするだけで、これは有料ではありません。
保持期間などの上限を伸ばす有料プランは後日提供します。

## ベンチマークを用意する

何を測るかが、検知できる回帰の範囲を決めます。
良いベンチマークは、決定的で、気にしている経路を通り、そこそこの規模があります。
何をどう測るかはプロジェクトしだいなので、書き方とプロジェクト別の例は「ベンチマークの書き方」にまとめています。

この登録編では、例として Rails アプリの起動（boot）を計ります。
ベンチマークは `bin/rails runner ""` だけです。
アプリをブートして空のスクリプトを走らせるので、ベンチ用のファイルを書かずにそのまま使えます。
次の節で、これをワークフローに置きます。

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
    branches: [ main, master ]   # ベース（デフォルトブランチ）を記録
  pull_request:                  # ベースと比較
jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read         # checkout
      id-token: write        # OIDC アップロード（シークレット不要）
      checks: write          # Check Run を書く（public）
      pull-requests: write   # sticky コメントを書く（public）
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bin/rails runner ""
```

`run:` には、プロファイルしたいコマンドだけを書きます。
Action がそれを rperf の下で計測し、プロファイルを書き出します。
自分で rperf コマンドを書きたいときは、入力 `record: false` を付けて `run:` に直接 `rperf record ...` を書きます。

`permissions` は四つ書きます。
`id-token: write` が無いと OIDC トークンが取れず、アップロードできないので、必ず付けてください。
`contents: read` は `actions/checkout` がリポジトリを取得するためです。
`checks: write` と `pull-requests: write` は、public で Action が `GITHUB_TOKEN` を使い Check Run と sticky コメントを書くためです（private では App が書くので無くても動きますが、書いておいて害はありません）。
`permissions:` を書くと、挙げていない権限は none になるので、四つとも明示します。

## 閾値とコメントの設定（任意）

閾値は、回帰したときに Check Run で警告（⚠️）を出すための仕組みです。
どこからを回帰とみなすか、その基準は指標ごとに自分で決められます。
基準は、base から head への各指標の増加（アロケーション、GC、時間など）の上限として与えます。
超えると Check Run に ⚠️ が付き、`comment` 設定に応じて PR コメントが出ます。
閾値は任意で、設定しなければ数字は出ますが、警告もコメントも付きません。
回帰したら警告してほしいときだけ設定します。

設定は全部ワークフローに書きます（別ファイルは不要）。
全体設定を job の `env` に一度だけ書き、必要ならベンチごとに上書きします。

```yaml
jobs:
  bench:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write, checks: write, pull-requests: write }
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
          run: bin/rails runner ""
```

閾値のキーと、まず設定するなら使いたい値は次のとおりです（prperf 自体に既定値はなく、設定して初めて効きます）。

| キー | 推奨デフォルト | 意味 |
|---|---|---|
| `alloc` | `"+10%"` | アロケーション数の増加。`"+5000"` のように絶対値でも書ける |
| `gc_count` | `"+2"` | GC 回数（minor+major）の増加 |
| `total_ms` | `"+20%"` | 実行時間の増加。ノイズが大きいので相対（%）で |
| `cpu_ms` | `"+15%"` | CPU 時間の増加 |
| `method` | （指定なし） | 名前を挙げたメソッドの self 占有率が、その % を超えたら。例 `{ "JSON.generate": "15%" }` |

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
| `run` | (必須) | プロファイルしたいコマンド。Action が rperf の下で計測する（コマンドは書き換えない） |
| `record` | `true` | `run` を rperf の下で計測する。`false` で自分の rperf コマンドを `run` に直接書く |
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
    run: bin/rails runner ""
- uses: rperf-dev/prperf-action@v1
  with:
    benchmark: render
    run: ruby bench/render.rb
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
GitHub は fork 起因のワークフローに `id-token: write` を与えず、ワークフロートークンも read-only なので、OIDC トークンが取れずアップロードできません。
同一リポジトリのブランチ PR は問題ありません。

アップロードの失敗（プラン上限、レート制限、サーバーエラー）は警告のみで、ステップは成功扱いです。
計測コマンド自体の失敗だけがステップを落とします。

無料β期間中は public も private も無料で使えます。
private は prperf の GitHub App をインストールして使います。
結果は 15 日間保持します（free プラン）。
保持期間などの上限を伸ばす有料プランは後日提供します。
