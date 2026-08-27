---
title: "クライアントツール - RDB(MySQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > クライアントツール（MySQL）


## クライアントツール（MySQL）

### Sequel Pro

- 通常版はMySQL バージョン8には対応していない(?)。
    - `brew install --cask homebrew/cask-versions/sequel-pro-nightly` でnightly版をインストールする。
    - (参考) [Sequel ProでMySQL 8に接続する](https://qiita.com/yasudanaoya/items/2112e13e2ba164bdf47c)
- 踏み台サーバー経由で接続したサーバーからデータベース接続する場合、configに設定しておけば簡単に行える(?)。
    - (参考) [Sequel Proで踏み台サーバー経由接続する方法](https://qiita.com/t-toyota/items/be854ebacb42f7560584)
