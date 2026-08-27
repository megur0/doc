---
title: "ロック・排他制御 - RDB(MySQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > ロック・排他制御（MySQL）


## ロック・排他制御（MySQL）

### 一貫性読み取り（consistent-read）

- (参考) [InnoDB の一貫性のない読み取り](https://dev.mysql.com/doc/refman/8.0/ja/innodb-consistent-read.html)
- トランザクション分離レベルがREPEATABLE READ（デフォルトのレベル）である場合は、同じトランザクション内のすべての一貫性読み取りで、そのトランザクション内の最初のこのような読み取りで確立されたスナップショットが読み取られる。
- 分離レベルがREAD COMMITTEDの場合は、トランザクション内の各一貫性読み取りで、独自の新しいスナップショットが設定され、読み取られる。
- 一貫性読み取りは、InnoDBがREAD COMMITTEDおよびREPEATABLE READ分離レベルでSELECTステートメントを処理する際のデフォルトモードである。


### 分離レベル

- (参考) [InnoDB Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- デフォルトはREPEATABLE READ。「同じトランザクション内の一貫した読み取りは、最初のSELECT読み取りによって確立されたスナップショットを読み取る」とされている。
- SERIALIZABLEにすると、Consistent Readはすべて悲観ロックに変換される。（IME）SELECTが全部ロックされて挙動に戸惑ったことがあるが、確認すると設定がSERIALIZABLEになっていた、というケースがあった。


### ロックの種類

MySQLの特徴として、ギャップロック、ネクストキーロックがある。

- レコードロック: インデックスレコードのロック。
- ギャップロック: インデックスレコード間にあるギャップのロック、または先頭のインデックスレコードの前や末尾のインデックスレコードのあとにあるギャップのロック。
- ネクストキーロック: インデックスレコードに対するレコードロックと、そのインデックスレコードの前にあるギャップに対するギャップロックとを組み合わせたもの。

- (参考) [InnoDB のロック方法](https://dev.mysql.com/doc/refman/5.6/ja/innodb-record-level-locks.html)


### ロック取得状況を確認する

```sql
select * from performance_schema.data_locks;
```

`LOCK_MODE`:

- `IX` インテンション排他ロック、`IS` インテンション共有ロック
    - `IS`や`IX`に対しては競合しないが、`S`や`X`には競合する。
    - 基本的にInnoDBがレコードロックする場合は、テーブルに対してインテンションロックとレコードに対してのレコードロックを行う。
    - したがって、1行にレコードロックしている場合、`select * from performance_schema.data_locks` で見ると2つのロックが確認できる。テーブルに`X`がかけられている場合は、`IX`も`IS`もできない。
- `S` 共有ロック
- `X` 排他ロック
- `REC_NOT_GAP`
    - レコードロック（`NOT GAP`と書いてあるのはギャップロックではないことを明示している）。例えば、排他レコードロックされている場合は「`X, REC_NOT_GAP`」という形で表示される。
- `GAP`
    - ギャップロック

`LOCK_DATA`: `LOCK_TYPE='RECORD'` の場合はロックされたレコードの主キー値、それ以外の場合はNULL。

ロックしているトランザクションを特定してkillする:

```sql
SELECT trx_rows_locked FROM information_schema.INNODB_TRX;  -- スレッドIDなどがわかる
show processlist;
KILL 19784;
```


### ロックの範囲

- インデックスやユニーク制約がついていないカラムに対してWHEREで条件指定して占有ロックをかけると、テーブルロックになってしまう。
- 複数条件の場合は、1つでもインデックスやユニーク制約がついている条件が入っていれば、対象の行のみの行ロックになる(?)。

- (参考) [MySQLのロックについて](https://www.wakuwakubank.com/posts/201-mysql-lock/)
- (参考) [MySQLロック関連メモ](https://taiga.hatenadiary.com/entry/2018/02/12/170109)


### ギャップロックが発生する場合

SQLが空振りした際に取得される(?)。例えば `BETWEEN 3 AND 6` で取得した際、テーブルに3, 5, 8しかない場合は6が空振りしているので、8までギャップロックが取られる。これはファントムリードを回避するためとされている。

(IME) 「仮に他の処理で8が挿入されても、次に `BETWEEN 3 AND 6` を行ったときには関係ないのでは」と最初は疑問に思ったが、MySQLはインデックスを使ってロックをかけているため、このような挙動になるらしい(?)。

- (参考) [MySQLのギャップロックについて (1) - Qiita](https://qiita.com/ham0215/items/99679d499869365446ec)
- (参考) [MySQLのギャップロックについて (2) - Qiita](https://qiita.com/kenjiszk/items/05f7f6e695b93570a9e1)

無効化したい場合は、トランザクション分離レベルをREAD COMMITTEDに変更する。

ギャップロックの目的は、REPEATABLE READでファントムリードを回避するため。ISO SQLの仕様では、REPEATABLE READにおいてファントムリードは許容されるが、InnoDBでは発生しないようになっている。これは、ギャップロックによって走査した範囲への挿入をロックすることで実現している。
