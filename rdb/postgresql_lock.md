---
title: "ロック・同時実行制御 - RDB(PostgreSQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > ロック・同時実行制御（PostgreSQL）


## ロック・同時実行制御（PostgreSQL）

### SQLやロックの確認

- (参考) [PostgreSQLのロック確認方法](https://kinosuke.hatenablog.jp/entry/2019/07/23/125300)
- (参考) [PostgreSQLロックの確認方法とデッドロックの解除](https://www.compiere-distribution-lab.net/2015/06/18/postgresql-0007-ロック-lock-の確認方法とデットロックの解除/)

接続中クライアントの確認:

```sql
select * from pg_stat_activity;
```

ロック状況の確認:

```sql
SELECT l.pid,l.granted,d.datname,l.locktype, l.tuple, l.relation,l.relation::regclass,l.transactionid,l.mode
FROM pg_locks l  LEFT JOIN pg_database d ON l.database = d.oid
WHERE  l.pid != pg_backend_pid()
ORDER BY l.pid;
```

`pg_locks`の各項目内容については以下を参照。

- (参考) [PostgreSQL: pg_locks](https://www.postgresql.jp/document/14/html/view-pg-locks.html)

#### pg_locksで確認できる情報について

このテーブルには「テーブルロック」と「未獲得の行ロック」しか含まれない。したがって、

- 獲得済みの行ロックは確認できない。
- ロックモード（「FOR UPDATE」「FOR NO KEY UPDATE」「FOR SHARE」「FOR KEY SHARE」）の区別がつかない。

- (参考) [pg_locksについて - Qiita](https://qiita.com/myzkyy/items/d456747f2c70a748587a)

#### サーバーログへのロック内容の出力

サーバーログにロック内容や待機状態も出力される(?)。

- (参考) [Row locking and locking on transactionid](https://masahikosawada.github.io//2018/09/04/Row-locking-and-locking-on-transactionid/)
- (参考) [PostgreSQLのロック確認方法](https://kinosuke.hatenablog.jp/entry/2019/07/23/125300)
- (参考) [ロック監視について - Qiita](https://qiita.com/behiron/items/571562ea33b8212a4c32)


### トランザクションの分離レベル

- (参考) [PostgreSQL: トランザクションの分離レベル](https://www.postgresql.jp/document/14/html/transaction-iso.html)
- MVCCが採用されているため、ロックに関わらずダーティリードはすべての分離レベルで発生しない。
- READ COMMITTEDがデフォルトの分離レベル。コミット前ではなく問い合わせの直前の最新バージョンを取得するため、ファントムリード、反復不能読み取り、直列化異常が発生する。
- REPEATABLE READ: ISO SQLではファントムリードは許容されるが、PostgreSQLでは発生しない。

分離レベルの構文:

```sql
SHOW TRANSACTION ISOLATION LEVEL
```

トランザクション開始コマンドと一緒に使う:

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```


### MVCC

MVCCの詳細は[トランザクション分離レベル](./rdb_isolation_level.md)を参照。

- (参考) [PostgreSQL: MVCC](https://www.postgresql.jp/document/14/html/mvcc-intro.html)
- (参考) [MVCC and GC in PostgreSQL](https://masahikosawada.github.io/2021/12/22/MVCC-and-GC-in-PostgreSQL/)
- (参考) [Heroku: PostgreSQL Concurrency](https://devcenter.heroku.com/ja/articles/postgresql-concurrency)
- トランザクションIDは40億（32ビット）が限界(?)。40億を超えるケースは定期メンテナンスが必要になる(?)。


### PostgreSQLコマンドではロックを自動的に取得する

ほとんどのPostgreSQLコマンドでは自動的にロックを取得するので注意が必要。

- (参考) [PostgreSQL: 明示的ロック](https://www.postgresql.jp/document/14/html/explicit-locking.html)
- 例えば、UPDATE、DELETE、およびINSERTコマンドは、ROW EXCLUSIVEを獲得する。
- 例えば、トランザクションブロックであるレコードを更新した際、同時に他のトランザクションが同じレコードの更新をする場合、もう一方は更新が完了するまで待機することになる。


### NOWAIT

- (参考) [PostgreSQL: SELECT](https://www.postgresql.jp/document/14/html/sql-select.html)
- `NOWAIT` は `FOR UPDATE` のみに利用できる。以下はすべてエラーとなる。

```sql
delete from users where id = 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa' NOWAIT
select from users where id = 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa' NOWAIT
select from users where id = 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa' for select NOWAIT
update users set name='dummy name' where id = 'bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb' nowait
```

#### SELECT FOR UPDATE以外でNOWAITするにはどうすればよいか

先に `SELECT FOR UPDATE NOWAIT` でロックをかけてから、その後各処理をすればよさそう(?)。

- (参考) [OKWave: NOWAITに関する質問](https://okwave.jp/qa/q2013048.html)


### 明示的なロック取得

`LOCK` コマンドを使用する。

- (参考) [PostgreSQL: LOCK](https://www.postgresql.jp/document/14/html/sql-lock.html)

```sql
LOCK TABLE user_setting;
```

モードの指定がないため、もっとも強力なACCESS EXCLUSIVEを取得する。

#### insertによるロックとユニーク制約の検証

RowExclusiveLockはかかるが、競合しない限りロック待ちは発生しない。

**パターン1: ユニーク制約のあるテーブルへinsert**

前提として `users` は `uid` でユニーク。

セッション1:

```sql
begin;
INSERT INTO users (name, uid) VALUES ('name', 'aaaaaaa');
select * from users;
(1 row)
```

セッション2:

```sql
select * from users;
(0 row)
INSERT INTO users (name, uid) VALUES ('name', 'aaaaaaa');
```

待機状態になる。ロック競合の検知において、ユニーク制約はちゃんと見ているようだ。`pg_locks`を確認すると、`users`でRowExclusiveLockがかかっている。

セッション1:

```sql
commit;
```

セッション2:

```sql
ERROR:  duplicate key value violates unique constraint "uniq__users__uid"
DETAIL:  Key (uid)=(aaaaaaa) already exists.
```

**パターン2: ユニーク制約のないテーブルへinsert**

前提として `comments` にはユニーク制約はない。

セッション1:

```sql
begin;
INSERT INTO comments (user_id, comment) VALUES ('cccccccc-cccc-cccc-cccc-cccccccccccc', 'aaaaaaa');
select * from comments;
(1 row)
```

セッション2:

```sql
select * from comments;
(0 row) /* コミットされていないためまだ見えない */
INSERT INTO comments (user_id, comment) VALUES ('cccccccc-cccc-cccc-cccc-cccccccccccc', 'aaaaaaa');
select * from comments;
(1 row)
```

待機状態にはならない。制約がなければロック競合にはならないようだ。`pg_locks`を確認すると、`comments`でRowExclusiveLockがかかっている。

セッション1:

```sql
select * from comments;
...
(2 rows) //　ファントムリード
```


### ロックモード

- (参考) [PostgreSQL: 明示的ロック](https://www.postgresql.jp/document/14/html/explicit-locking.html)
- 行ロックモードの発動条件
    - FOR UPDATE: `SELECT FOR UPDATE`文、unique keyを更新するDELETE/UPDATE文
    - FOR NO KEY UPDATE: `SELECT FOR NO KEY UPDATE`文、unique keyを更新しないDELETE/UPDATE文
    - FOR SHARE: `SELECT FOR SHARE`文
    - FOR KEY SHARE: `SELECT FOR KEY SHARE`文
    - (参考) [ロックモードについて - Qiita](https://qiita.com/zyake/items/5f26c6227ff436a56e2c)


### ロッキングリードは集約関数と同時に利用できない

```sql
/*
これはエラーになる。(FOR UPDATEは集約関数と同時に利用できない)
SELECT COUNT(*) FROM comment_favorites WHERE user_id = 'dddddddd-dddd-dddd-dddd-dddddddddddd' FOR UPDATE;
*/
```


### ロックの検証をしてみる

以下は`BEGIN`内で実行（単なる実行だと即時終了して確認できないため）。

単なるSELECT文を実行:

```sql
select * from users where id = '10101010-1010-1010-1010-101010101010'
```

`pg_locks`を確認すると、relationに対するAccessShareLock（ACCESS EXCLUSIVEロックモードとのみ競合）が確認できる。

「FOR UPDATE（FOR NO KEY UPDATE）」「FOR SHARE」を実行してみる:

```sql
select * from users where id = '10101010-1010-1010-1010-101010101010' for update;
select * from users where id = '10101010-1010-1010-1010-101010101010' for share;
update users set name='taro' where id = '10101010-1010-1010-1010-101010101010';
```

`pg_locks`を確認すると、relationに対するRowShareLock（EXCLUSIVEおよびACCESS EXCLUSIVEロックモードと競合）が確認できる。

別のpidからリソースを取得しようとしてみる:

```sql
select * from users where id = '10101010-1010-1010-1010-101010101010' for share nowait;
-> could not obtain lock on row in relation "users"
```

元pidの方の処理がFOR SHAREのSQLの場合はエラーにならない。

#### インデックスを張っていない列の検証

MySQLは、インデックスが貼られていない列に対してのWHEREはテーブルロックとして共有ロックや排他ロックがかかってしまう（詳細は[ロック・排他制御（MySQL）](./mysql_lock.md)参照）。

- (参考) [MySQLとのロックの違いについて](https://yk5656.hatenadiary.org/entry/20140720/1406238608)

PostgreSQLは大丈夫そう(?)。

```sql
begin;
select * from users where name = 'jiro' for share;
(他のpidで)
update users set name='taro' where name='jiro'; # これは当然待機状態になる
update users set name='taro' where name='saburo'; # これは即時完了
```


### ロッキングリードは取得した行しかロックできない

以下は例。

```sql
/* トランザクションでロック  */
BEGIN;
/* (X) */SELECT * FROM comment_favorites WHERE user_id = 'dddddddd-dddd-dddd-dddd-dddddddddddd' FOR UPDATE;
ROLLBACK;

/* (X)の直後のタイミングで別のセッション(コネクション)で以下を実行する */
/* insert: これはエラーにならない */
insert into comment_favorites (user_id, from_user_id, comment, comment_id) values ('dddddddd-dddd-dddd-dddd-dddddddddddd', 'eeeeeeee-eeee-eeee-eeee-eeeeeeeeeeee', 'test', 'ffffffff-ffff-ffff-ffff-ffffffffffff');
/* update: これはロック待ちとなる */
UPDATE comment_favorites SET comment='' WHERE user_id = 'dddddddd-dddd-dddd-dddd-dddddddddddd';
/* select for update: これは即時エラーとなる */
SELECT * FROM comment_favorites WHERE user_id = 'dddddddd-dddd-dddd-dddd-dddddddddddd' FOR UPDATE NOWAIT;
```


### INSERTに対してロックをかけるにはどうするか

上記の通り、ロッキングリードでは「まだ存在しない行」はロックできない。そもそもINSERTは一意インデックス以外ではブロックされない。

- (参考) [PostgreSQL: INSERT](https://www.postgresql.jp/document/14/html/sql-insert.html)

> 一意インデックスのないテーブルへのINSERTは同時実行中の処理によりブロックされることはありません。

#### 方法1: 勧告的ロック用関数を利用する

- (参考) [勧告的ロックについて](https://zenn.dev/link/comments/da2eadbef468cf)
- (参考) [PostgreSQLのアドバイザリーロック - Zenn](https://zenn.dev/mpyw/articles/rdb-advisory-locks)
- ロック専用のテーブルを設けて利用する方法もあるが、PostgreSQLが提供する関数を利用するのが最も簡単だろう(IMO)。
- 勧告的ロック用関数
    - (参考) [PostgreSQL: 管理用関数](https://www.postgresql.jp/docs/9.4/functions-admin.html)

検証:

```sql
/* トランザクションでロック  */
begin;
/* (X) */SELECT pg_try_advisory_xact_lock(hashtext('comments_dddddddd-dddd-dddd-dddd-dddddddddddd'));
rollback;

/* (X)の直後のタイミングで別のセッションで実行  */
SELECT pg_try_advisory_xact_lock(hashtext('comments_33333'));/* これは通る */
SELECT pg_try_advisory_xact_lock(hashtext('comments_dddddddd-dddd-dddd-dddd-dddddddddddd'));/* これは通らない */
```

#### 方法2: unique制約を利用する

例えば、「同じuser_idのレコードはXX個まで登録可能」とする仕様の場合を考える。

- 何番目かを表す列（`number`とする）を追加する。
- 複合インデックス（`user_id`, `number`）を作成する。
- プログラム側で、現在の`user_id`の全データを取得して、`number`について1〜XXの範囲で空きのあるものがあればinsertする。ない場合はエラーとする。
- 同時のトランザクションで同じ`user_id`, `number`に対してinsertが発生しても、ユニーク制約によってエラーとなる。

デメリットは実装が面倒であること。


### ロックによる障害の例

- (参考) [ロックによる障害事例 - Qiita](https://qiita.com/YujiSoftware/items/7baec688d046796264f9)
    - 手作業で `BEGIN; SELECT * FROM user_setting WHERE xxx = 1;` を実行した（ACCESS SHARE）。
    - `LOCK TABLE user_setting` を行う既存処理が存在した（つまりACCESS EXCLUSIVE）。この処理が手作業によるロック解除待ちになった。このロック解除待ちによって、他のSELECT処理も待機になる。
    - 結果、1つのスレッドが`LOCK TABLE`で、多数のスレッドがSELECTで止まってしまい、データベースとのコネクションプールが枯渇。システムダウンした。


### ALTER TABLEによるロック

副構文によって異なるが、ほとんどはテーブルに対してACCESS EXCLUSIVEを取得する。

- (参考) [PostgreSQL: ALTER TABLE](https://www.postgresql.jp/document/12/html/sql-altertable.html)

> 要求されるロックレベルはそれぞれの副構文によって異なることに注意してください。特に記述がなければACCESS EXCLUSIVEロックを取得します。


### （検証）分離レベルとロックの範囲

自身が明示的にロックした行以外に対しては悲観ロックはかからない。取得する行について、分離レベルに応じてファントムリードが発生するかどうかは変わる。

#### READ COMMITTEDで、ファントムリードが起きることを確認

```sql
begin;
select * from users where uid in ('sample_uid_a') for update;
(0 rows)
```

ここで、別のpidでinsertする。

```sql
INSERT INTO "users" ("uid") VALUES ('sample_uid_a');
```

```sql
select * from users where uid in ('sample_uid_a') for update;
(1 row) // ファントムリード
```

#### SERIALIZABLEでファントムリードが起きないことを確認

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
select * from users where uid in ('sample_uid_a') for update;
(0 rows)
```

ここで、別のpidでinsertする（※ ここはロックされない。MySQLと違ってギャップロックがない感じ(?)）。

```sql
INSERT INTO "users" ("uid") VALUES ('sample_uid_a');
```

```sql
select * from users where uid in ('sample_uid_a') for update;
(0 rows) // ファントムリードは起きない
```

#### 明示的にロックした行がロックされていることを確認

```sql
begin;
select * from users where uid in ('sample_uid_a') for update;
```

別pid:

```sql
delete from users where uid in ('sample_uid_a'); // 待機になる
```

#### 明示的にロックしていない行はロックされないことを確認（LIMITのケース）

```sql
begin;
BEGIN
select * from users where uid in ('sample_uid_0001', 'sample_uid_0002') for update LIMIT 1;
...
(1 row)
```

別のpidで以下を実行する。

```sql
select * from users where uid in ('sample_uid_0001') for share nowait;
ERROR:  could not obtain lock on row in relation "users"
select * from users where uid in ('sample_uid_0002') for update nowait;
...
(1 row) // 取得できる
```

SERIALIZABLEでも結果は同じ。


### PostgreSQLはDDLもロールバックの対象

したがって、CREATEやALTERなどもロールバックされる。TRUNCATEもロールバックされるようだが、副作用等がどのように働くのかは未確認(TODO)。

- (参考) [PostgreSQLのTRUNCATE内部実装](https://dev.classmethod.jp/articles/postgresql-internal-truncate/)

OracleやMySQLではロールバックされない。
