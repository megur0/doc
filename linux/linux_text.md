---
title: "テキスト処理(sed・awk・jq) - Linux"
updated: 2026-08-29
---

[TOP(About this memo))](../README.md) > [一覧(Linux)](./README.md) > テキスト処理(sed・awk・jq)

## jq
* JSONを整形して見やすく表示できる。`curl` の出力をパイプする用途などで便利。
```
echo '{"errors":[{"message":"The user credentials were incorrect.","extensions":{"category":"authentication"},"locations":[{"line":2,"column":3}],"path":["login"]}]}' | jq
```
* キーの抽出もできる。
```
cat response.json | jq '.errors[0].message'
```

## sed
* (参考) https://www.atmarkit.co.jp/ait/articles/1610/18/news008.html
* 主なオプション
    * `-r`（GNU sedでは `-E` も同義） スクリプトで拡張正規表現を使用する
        * BSD系（macOS標準）のsedは `-r` が使えず `-E` を使う。両対応させたい場合は `-E` の方が無難。
    * `-n` 出力コマンド以外の出力を行わない（デフォルトでは、処理しなかった行はそのまま出力される）
    * `-i` ファイルを直接上書きする
        * GNU sedは `sed -i 's/aaa/AAA/' test.txt`、BSD系は `sed -i '' 's/aaa/AAA/' test.txt` のように空文字のバックアップ拡張子が必要で、書き方が異なる点に注意。

### 置換
* `sed -e 置換コマンド 入力ファイル`
    * `-e` は省略できる。ただし省略した場合、sedの第一引数がコマンド、第二引数以降が入力ファイルと解釈される。
* `aaa` を `AAA` に置換する
    * `sed 's/aaa/AAA/g' test.txt`
        * 先頭の `s` は substitute（置換）コマンドを意味する。
        * 末尾の `g` は、マッチした文字列を全て置換することを意味する。`g` がなくても全行に対して置換は実行されるが、1行に2つ以上マッチした場合は1つ目しか置換されない。
    * `sed '3,5s/abc/ABC/g' test.txt` … 3行目から5行目のみを置換の対象とする
    * `sed '3s/abc/ABC/g' test.txt` … 3行目のみを置換の対象とする
* 上書き
    * `sed -i 's/aaa/AAA/' test.txt`
* マッチした行全体を `&` で参照できる
    * `ls | sed -E "s/(.*)\.jpg$/& \1.jpeg/"`
        * `test.jpg` に対して `test.jpg test.jpeg` のような出力になる。
* `$PATH` を見やすく分割する
    * `echo $PATH | sed -e 's/:/\n/g'`
* sedは基本的に行単位で処理するため、複数行にまたがるパターンを一度に置換するのは素直に書けない。行を絞ってからパイプで繋ぐ方が分かりやすい。
    ```
    echo "$INFO" | sed -n 6p | sed -E 's/^.*ID: (.*)$/CLIENT_ID=\1/' >> .env
    echo "$INFO" | sed -n 7p | sed -E 's/^.*secret: (.*)$/CLIENT_SECRET=\1/' >> .env
    ```
    * なお、sedの正規表現（BRE/ERE）には `*?` のような最短一致の指定がない。最短一致が必要な場合はperlやawk、あるいは対象を絞る書き方に置き換える。

### 削除
* `sed '3,5d' test.txt` … 3〜5行目を削除
* `sed 's/ //g' test.txt` … 空白を削除
* `sed '/^$/d' test.txt` … 空行を削除
* `sed '/^#/d' source.txt` … `#` で始まるコメント行を削除
* `cat source.txt | grep -v '^#' | sed '/^$/d'` … 先頭 `#` の行と空行を削除

### 行の追加
* `sed '3i aaaaaaaaa' test.txt` … 3行目の前に挿入
* `sed '3a aaaaaaaaa\nbbbbbbbbb' test.txt` … 3行目の後に追加

### その他
* sedの結果を使ってコマンドを組み立てる例
    * `ls | sed -En "s/(.*)\.jpg$/mv & \1.jpeg/p"`
    * 出力を確認した上で `| sh` に渡す、という使い方ができる。
* (参考) https://ez-net.jp/article/7A/9qV1-XpJ/6o8QiIwz08um

## awk
* awkはテキストの各行に対して `pattern { action }` を実行する。この基本形さえ押さえれば大きく間違えない。
* (参考) https://hfuji.hatenablog.jp/entry/2017/08/17/194239
* 例）空白区切りの2列目だけを表示する
```
ps aux | awk '{print $2}'
```
* 例）特定の列を数値としてフォーマットして表示する
```
ps alx | awk '{printf ("%d\t%s\n", $8, $13)}' | sort -nr | head -10
```
