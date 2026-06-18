# このサービスとは

## ひとことで

prperf は、コードの変更で性能が悪化していないかを PR ごとに自動チェックして PR に知らせる GitHub App です。
測定はあなたの CI の中で OSS の Ruby プロファイラ [rperf](https://github.com/ko1/rperf) が行います。
prperf 自身は、マージ先ブランチ(ふつう main)の最新の計測と、この PR の計測を比べて結果を通知するだけの薄い App です。

このマージ先ブランチ側の基準を base、PR 側を head と呼びます(GitHub の PR 用語と同じ)。
本マニュアルでは以降この呼び方を使います。

> テストカバレッジを CI で追う Codecov を知っていれば、その「性能版」と思ってください。

rperf は時間(CPU)ベースのサンプリングプロファイラで、本体は「どこで時間を使ったか」を表すフレームグラフです。
prperf はそのプロファイルから実行時間、GC、アロケーションを取り出し、base と head で比較します。
PR を作ると、Check Run にこんな要約が出ます。

> 2,001ms → 2,140ms (+7%) · alloc 48,741 → 59,950 (+23%) · GC 4 → 7

このコミットで性能がどう変わったかを、マージ前の PR の時点で気づけるようにするのが prperf です。

## 何をするか、しないか

する:

- PR ごとに base から head への性能差(アロケーション、GC、時間)を Check Run に表示
- 閾値を超えたときだけ PR にコメント(sticky、通知は静か)
- フレームグラフの diff で「どのメソッドが重くなったか」を可視化

しないこと:

- 本番監視ではありません。
  Datadog や Grafana の代替ではなく、補完です(本番の継続監視はそちら、PR 時点の回帰検知が prperf)。
- CI を落としません。
  判定はあくまで参考表示で、Check の conclusion は常に success です。
- あなたのコードをサービス側で実行しません。
  測定はあなたの CI の中で行われ、prperf はその結果(プロファイル)を受け取って比較するだけです。
  これが「薄い App」として成立する理由で、セキュリティとコストの両面で軽くなります。

つまり prperf は、計測と判定を CI 側に置き、サービス側は比較と通知だけを担当します。

## 全体像

```
あなたの CI (GitHub Actions)
  └─ prperf-action
       ├─ rperf でベンチを N 回計測
       └─ プロファイル(.json.gz)を prperf サーバーへアップロード
            │  (GitHub OIDC トークンで認証 → シークレット設定不要)
            ▼
prperf サーバー
  ├─ base と head を比較
  └─ Check Run / PR コメントに結果を通知
```

この構成では CI が計測を実行し、prperf サーバーは受け取ったプロファイルを base と head の組として比較します。

## 使う側の体験

1. GitHub App をリポジトリにインストール
2. 提供する GitHub Action をワークフローに数行追加
3. PR を作ると Check と PR コメントに結果が付く

ワークフローは 1 本です。
push(既定ブランチ)で base を記録し、pull_request でその base と比較します。
base か head かは、prperf が OIDC トークンの ref から判定します。

```yaml
# .github/workflows/prperf.yml
name: prperf
on:
  push:
    branches: [main, master]   # base を記録（既定ブランチ）
  pull_request:                # PR を base と比較
jobs:
  bench:
    runs-on: ubuntu-latest
    permissions: { contents: read, id-token: write }
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with: { bundler-cache: true }
      - uses: rperf-dev/prperf-action@v1
        with:
          run: bundle exec ruby bench/run.rb
```

## なぜ信頼できるか(設計の勘所)

判定では、実行時間よりもアロケーション数と GC 回数を重視します(CI の実行時間は ±10〜20% ブレるためです)。
rperf が測る中心は時間(フレームグラフ)ですが、その時間は参考にとどめます。
結果には時間、GC、アロケーションのすべてと、フレームグラフが出ます。

シークレットは要りません。
認証は GitHub Actions の OIDC トークンで、API キーの発行や管理が不要です。

通知は静かです。
Check Run は常設の置き場で通知ゼロ、コメントは閾値超過時のみ、PR につき 1 通を編集します。

重くなった理由まで分かります。
フレームグラフ diff で、重くなったメソッドを特定できます。

## 誰のためか

性能回帰を PR で止めたい、public な gem やライブラリの作者に向いています。
依存の更新やリファクタで、気づかないうちにアロケーションや起動が重くなるのを、PR の時点で止められます。

性能が UX や売上に直結する private な Rails アプリのチームにも向いています。
重くなる変更を、本番に出る前に、レビューと同じ場所で捕まえられます。

料金は、public リポジトリが無料、private が有料プランです（現在は無料βのため public のみ）。

## このマニュアルの読み進め方

- 登録編：インストールとワークフロー追加
- ベンチマークの書き方、Rails クイックスタート：何を測るか
- 読み方編、フレームグラフの読み方：結果の読み取り

次は「登録編」へ。
