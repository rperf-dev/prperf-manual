# Book

📖 **公開版（GitHub Pages）**: https://rperf-dev.github.io/prperf-manual/

この本は [ligarb](https://github.com/ko1/ligarb) で生成しています。

## フィードバック

本文の誤り・わかりにくい点・疑問は [Issue](https://github.com/rperf-dev/prperf-manual/issues/new?template=book-feedback.yml) からどうぞ。
公開ページでは本文を選択して「Report as issue」からも送れます。

## ローカルでビルド

```bash
ligarb build   # build/index.html を生成
ligarb serve   # ローカルプレビュー
```

セットアップ手順は [SETUP.md](SETUP.md) を参照してください。

## 継続的ベンチマーク

この本を生成する `ligarb build` の所要時間は、PR ごとに [prperf](https://rperf.atdot.net)
で計測し、base（main）と比較しています（prperf 自身のドッグフーディング）。結果は各 PR の
Check Run に表示されます。

計測は1コミットあたり既定で **3 回**実行し（[prperf-action](https://github.com/rperf-dev/prperf-action)
の `count` 入力で変更可能）、**中央値**で比較・表示します。初回 run はコールドスタート（Ruby 起動・
ライブラリ読み込み・キャッシュ未温）で遅くなりがちなので、中央値で外れ値を吸収する設計です。ビューアの
サイドバーも各コミット 1 行（中央値）で表示します。
