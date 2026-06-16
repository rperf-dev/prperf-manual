# 登録編

prperf を使い始めるまでの流れです。所要 10〜15 分。サービスの概要は
「このサービスとは」を参照してください。

やることは 3 つです。

1. GitHub App をインストール
2. ワークフローを追加(PR 用・main push 用の 2 本)
3. ベンチマークを用意

## 1. 前提

- **rperf 0.10 以上**を Gemfile に入れていること。prperf はプロファイルに
  埋め込まれた `meta` / `summary` を使います。古い rperf だと action が
  明確なエラーで止まります。
- 計測対象の**ベンチマークコマンド**があること(後述)。
- 対象は **public リポジトリ**。private リポジトリは有料プラン(現在は無料β
  のため public 専用)。

## 2. GitHub App をインストール

prperf の GitHub App ページからリポジトリにインストールします。これにより
prperf がそのリポジトリの Check Run と PR コメントを書けるようになります。

## 3. ワークフローを追加

prperf は **PR の head** と **base ブランチの最新スナップショット**を比較します。
そのため **2 本**のワークフローが要ります。

- **PR 用**: PR のたびに計測してアップロード(比較される側)
- **main push 用**: 既定ブランチへの push で計測(比較の **base** を供給)

main 用が無いと比較対象が無く「比較対象なし、今回の数値のみ」になります。

### PR 用ワークフロー

```yaml
# .github/workflows/prperf.yml
name: prperf
on: pull_request

jobs:
  bench:
    runs-on: ubuntu-latest
    permissions:
      contents: read
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

`permissions: id-token: write` を**必ず**付けてください。これが無いと OIDC
トークンが取れず、アップロードできません。

### main push 用ワークフロー

```yaml
# .github/workflows/prperf-base.yml
name: prperf (base)
on:
  push:
    branches: [ main ]

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

`run:` は両方で同じにします(同じものを計測しないと比較になりません)。

## 4. ベンチマークを用意する

ここが一番大事で、一番手間のかかるところです。**何を測るか**で価値が決まります。

`run:` に渡すのは「rperf でプロファイルを 1 本以上吐くコマンド」です。
`$PRPERF_DIR` は action が用意する出力先で、`--snapshot-dir "$PRPERF_DIR"` に
渡すのが定番です。

良いベンチの条件:

- **決定的**であること。乱数・時刻・ネットワーク・外部 I/O に依存しないほど
  良い(アロケーションや GC 回数がブレない)。
- **気にしている経路を通る**こと。ベンチが触らないコードの PR は数字が
  動かず、Check は毎回「変化なし」になります。
- **そこそこの規模**。一瞬で終わるベンチはサンプル数が少なく不安定です。

### ゼロ設定で始めたいなら(Rails の boot 計測)

意味のあるベンチをまだ書けない場合、**起動(boot)の計測**はどの Rails
アプリでも動き、決定的です。eager load や gem 追加による起動劣化という実在の
回帰を捕まえられます。

```yaml
run: bundle exec rperf record --snapshot-dir "$PRPERF_DIR" -- bin/rails runner ""
```

まずこれで「数字が出る」体験を作り、徐々に本物のベンチへ移行するのがおすすめ
です。

## 5. 閾値とコメントの設定(任意)

閾値は**任意**です。設定しなければ Check Run に数字は出ますが ⚠️ もコメントも
付きません。「回帰したら警告してほしい」ときだけ設定します。

設定は**全部ワークフローに書きます**(別ファイルは不要)。**全体設定**を job の
`env` に一度だけ書き、必要なら**ベンチごとに上書き**します。

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

閾値のキー:

| キー | 値の例 | 意味 |
|---|---|---|
| `alloc` | `"+10%"` / `"+5000"` | アロケーション数の増加(相対/絶対) |
| `gc_count` | `"+2"` | GC 回数(minor+major)の増加 |
| `total_ms` | `"+20%"` | 実行時間(ノイズが大きいので相対推奨) |
| `cpu_ms` | `"+15%"` | CPU 時間 |
| `method` | `{ "JSON.generate": "15%" }` | メソッドの self 占有率の絶対値超過 |

- 値は summary 系が `"+N%"`(相対)か `"+N"`(絶対)、method 系が `"N%"`。
- 不正な値は無視され、Check Run に警告が 1 行出ます(CI は落ちません)。
- 相対閾値(`+10%`)はベンチをまたいで素直に機能します。絶対閾値や method は
  ベンチごとに意味が変わるので、必要なときだけベンチ別に上書きしてください。

### コメントの出し方

`comment` 入力で制御します(既定 `on_threshold`)。

| 値 | 挙動 |
|---|---|
| `on_threshold` | 閾値超過時のみ PR コメント(既定) |
| `always` | 毎回コメント |
| `never` | コメントしない(Check Run だけ) |

コメントは PR につき 1 通で、push のたびに**同じコメントを編集**します(通知が
増えません)。

## 6. action の入力一覧

| 入力 | 既定 | 説明 |
|---|---|---|
| `run` | (必須) | 計測コマンド。`.json.gz` を 1 本以上吐くこと |
| `count` | `3` | 計測回数。サーバーは中央値で比較 |
| `benchmark` | `default` | ベンチ系列名。1 コミットで複数ベンチを独立比較できる |
| `thresholds` | `""` | このベンチの閾値(全体設定をキー単位で上書き) |
| `comment` | `on_threshold` | コメントの出し方 |
| `server` | `https://rperf.atdot.net` | prperf サーバー(差し替え可) |
| `upload` | `true` | `false` で計測のみ(アップロードしない) |

## 7. 複数ベンチマーク

1 コミットを複数のベンチで測れます。**ステップを分けて**それぞれに違う
`benchmark` 名を付けるだけ。サーバーは各ベンチを自分の base と比較し、**1 つの
Check Run** にまとめて表示します。

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

PR 用と main push 用で**同じ benchmark 名**を使ってください(各系列に base が
要るため)。

## 8. 動作確認

1. まず main に push → main 用ワークフローが走り、base スナップショットが
   サーバーに届きます。
2. 適当な PR を作る → PR 用ワークフローが走り、**Check Run に数字**が出れば成功。
3. アップロード結果のリンクは各ジョブの **Summary** にも出ます。

## 制約

- **fork からの PR は計測できません**。GitHub は fork 起因のワークフローに
  `id-token: write` を与えないため、OIDC トークンが取れません。同一リポジトリの
  ブランチ PR は問題ありません。
- アップロードの失敗(プラン上限・レート制限・サーバーエラー)は**警告のみ**で、
  ステップは成功扱い。計測コマンド自体の失敗だけがステップを落とします。
- 無料β期間中は **public リポジトリのみ**。private は有料プランで近日提供予定。
