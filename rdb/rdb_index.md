---
title: "インデックス - RDB"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > インデックス


## インデックス

### インデックスとは

- (参考) [インデックスの基本 - Qiita](https://qiita.com/towtow/items/4089dad004b7c25985e3)
- ソートされた情報をDBが保持してくれるので、探索の際に全探索ではなく2分探索等ができ、高速になる。
- 一般的にカーディナリティが低いカラム（値の取りうる範囲が少ない）につけても効果は低い。
- 複合インデックスの場合、カーディナリティが低いカラムを後ろに持っていく方がよい。
    - ただし複合インデックスの場合でも、カーディナリティが低いカラムはそもそも入れるべきではないケースもある(?)。
        - (参考) [DBインデックスの考え方](https://www.gatc.jp/gat/it/it02dbindex.html)
    - また、使い方に応じて順番を適切にすることも大事（例: `code, category` だと、カテゴリ単体で検索をしたときには `category, code` の方がベター、という話）。
        - (参考) [複合インデックスの列順について](https://tech.excite.co.jp/entry/2021/04/27/150029)


### B-tree

- MySQLではB+treeが採用されている。B-treeをより発展させたもの。
    - (参考) [B-treeとB+treeの違い - Qiita](https://qiita.com/gakinchoy7/items/f4eed87157ceb46fb7e9)


### 複合インデックス

複合でソートされた情報を持つ。

注意点として、1つ目に指定したカラムは単体で検索してもインデックスとして有効だが、2つ目以降で指定したカラムは単体で見た場合には有効なソート情報にならない。

- (参考) [複合インデックスの注意点 - Qiita](https://qiita.com/keii1111/items/02861c74bcd57ab1e290)


### SQLの検査と最適化

- `EXPLAIN` で `possible_keys`、`key` の値を確認できる。
    - `possible_keys` に存在していても `key` になければ、実際にはインデックスとして使われていない。
- (IME) `ORDER BY created_at` があったため `created_at` にインデックスをつけたが効かなかった事例がある。`LIMIT 10` をつけたところインデックスが効いた。WHEREやLIMITによる絞り込みがあることで、インデックスが効くケースがある(?)。
