# prperf マニュアル

prperf は、コードの変更で性能が悪化していないかを PR ごとに自動チェックする薄い GitHub App です。
測定は CI のなかで OSS の Ruby 向けサンプリングプロファイラ [rperf](https://github.com/ko1/rperf) が行い、prperf は base（main など）とこの PR を比べて結果を PR に通知するだけです。
テストカバレッジを CI で追う Codecov を知っていれば、その性能版にあたります。

PR を作ると、Check Run に次のような数字が出ます。

> 2,001ms → 2,140ms (+7%) · alloc 48,741 → 59,950 (+23%) · GC 4 → 7

全体像は次章「このサービスとは」で説明します。

## prperf ひとめぐり

### 導入

1. ベンチマークを用意します。ここでは `bin/rails runner ""` でブート時間を計るベンチマークとします。
2. それを実行するワークフローを追加します。`push`（既定ブランチ）と `pull_request` の両方をトリガにします。

public リポジトリはこれだけで動きます。
private リポジトリは加えて prperf の GitHub App をインストールします（有料プラン）。

```yaml
# .github/workflows/prperf.yml
name: prperf
on:
  push:
    branches: [main, master]   # base を記録（既定ブランチ。両方書けば main でも master でも動く）
  pull_request:                # PR を base と比較
jobs:
  bench:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write, checks: write, pull-requests: write }
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with: { bundler-cache: true }
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bin/rails runner ""   # ← 計測コマンド（手順1のベンチ）
```

閾値による警告、複数ベンチマーク（`benchmark`）、コメントの制御（`comment`）、計測回数（`count`、既定 3 回、中央値）といった設定もあります（詳しくは「登録編」）。

### 結果

各 PR では、GitHub の PR 画面の Checks にそのまま結果が出ます（base と比べた要約）。
閾値を超えたときだけ PR にコメントが付きます。
どのメソッドが重くなったかは、フレームグラフの diff でわかります（詳しくは「読み方編」）。

PR と push のたびに結果が記録され、[prperf.atdot.net](https://prperf.atdot.net) でこれまでの履歴（推移）を確認できます。

prperf は CI を落とさず、シークレットも要りません。
ただし fork からの PR は計測できず、無料βの間は public リポジトリだけが対象です。
