---
title: "コマンド - RDB(PostgreSQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > コマンド（PostgreSQL）


## コマンド（PostgreSQL）

### prepared statement

- (参考) [PostgreSQL: PREPARE](https://www.postgresql.org/docs/current/sql-prepare.html)

```sql
PREPARE fooplan (int, text, bool, numeric) AS
    INSERT INTO foo VALUES($1, $2, $3, $4);
EXECUTE fooplan(1, 'Hunter Valley', 't', 200.00);
```

#### prepared statementの一覧

```sql
select * from pg_prepared_statements;
```

ただし、現在接続しているセッションしか見ることはできない。

- (参考) [How to list all prepared statements for all active sessions - Stack Overflow](https://stackoverflow.com/questions/12159422/how-to-list-all-prepared-statements-for-all-active-sessions)

#### 解除

- `DEALLOCATE 〜`
    - (参考) [PostgreSQL: DEALLOCATE](https://www.postgresql.jp/document/14/html/sql-deallocate.html)
    - プリペアド文を明示的に割り当て解除しなかった場合、セッションが終了した時に割り当てが解除される。
    - `DEALLOCATE ALL` で全部のプリペアド文が解除される(?)。
- `DISCARD 〜`
    - (参考) [PostgreSQL: DISCARD](https://www.postgresql.jp/document/14/html/sql-discard.html)
    - `DISCARD PLANS` でもよさそう(?)。

#### キャッシュを無効化

`preparedStatementCacheQueries=0` とするとキャッシュがなくなる。ただし、パフォーマンス的に0にするのはあまりよくない(IMO)。


### psql

```
psql -U postgres -d データベース名
\dt;
\d users;
```


### EXPLAIN

`ANALYZE` を使うと処理も実際に実行されるので注意。

```sql
EXPLAIN (analyze true, format json) select * from users WHERE uid = 'sample_uid_a';
EXPLAIN (analyze true, format json) insert into users (uid) values('sample_uid_b');
EXPLAIN (analyze true, format json) update users set uid = 'sample_uid_c' WHERE uid = 'sample_uid_a';
```
