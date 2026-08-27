---
title: "SQL文法 - RDB(PostgreSQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > SQL文法（PostgreSQL）


## SQL文法（PostgreSQL）

### ダブルクオテーション（二重引用符）

- (参考) [PostgreSQL: SQL構文](https://www.postgresql.jp/document/8.1/html/sql-syntax.html)
- カラムやテーブル名などをダブルクオテーションで囲むと、通常は使えない予約語（SELECTなど）や特殊文字を使うことができる。また大文字と小文字が区別される。ダブルクオテーションを使わない場合は、名前は常に小文字に解釈される。
- ダブルクオテーションで囲むか、囲まないかは統一しておいた方がベター。

> 引用符が付かない名前は常に小文字に解釈されますが、識別子を引用符で囲むことによって大文字と小文字が区別されるようになります。例えば、識別子FOO、foo、"foo"はPostgreSQLによれば同じものとして解釈されますが、"Foo"と"FOO"は、これら3つとも、またお互いに違ったものとして解釈されます（PostgreSQLが引用符の付かない名前を小文字として解釈することは標準SQLと互換性がありません。標準SQLでは引用符の付かない名前は大文字に解釈されるべきだとされています。したがって標準SQLによれば、fooは"FOO"と同じであるべきで、"foo"とは異なるはずなのです。もし移植可能なアプリケーションを書きたいならば、特定の名前は常に引用符で囲むか、あるいはまったく囲まないかのいずれかに統一することをお勧めします）。

(IMO) 大文字と小文字が区別されるため、常に囲んでおく方がよい気もする。


### よくマイグレーションで使うSQLのメモ

- (参考) [PostgreSQL: ALTER TABLE](https://www.postgresql.jp/docs/14/sql-altertable.html)

```sql
ALTER TABLE "my_table" DROP CONSTRAINT uniq__my_table__user_id;
ALTER TABLE "my_table" ADD COLUMN "number" VARCHAR(500);
ALTER TABLE "my_table" ADD CONSTRAINT "uniq__my_table__user_id__number" UNIQUE("user_id", "number");
ALTER TABLE my_table ALTER COLUMN number SET NOT NULL;
```


### キャスト

`::` はキャスト。

```sql
SELECT now()::date
SELECT CAST(now() AS date)
```


### MySQLとの違い

- ギャップロックがないこと。
- 新しいカラムの追加時に列の位置を指定できない。
    - PostgreSQLにはこの機能がない。そもそも列の位置はRDBにとって重要ではなく、GUIツールが単に表示する順番がそうなっているだけとのこと(?)。
    - (参考) [How to add a new column after the Nth column - Stack Overflow](https://stackoverflow.com/questions/1243547/how-to-add-a-new-column-in-a-table-after-the-2nd-or-3rd-column-in-the-table-usin)
- クオテーションの違い
    - これだとOK: `insert into users ("uid", "name", "is_active", "external_id") values ('dhakfjkaf', 'test tarou', 1, 'dhilajfakja');`
    - これだとNG（`` ` `` は使えず、`"` はカラムだとみなされる）: `insert into users (`uid`, `name`, `is_active`, `external_id`) values ("dhakfjkaf", "test tarou", 1, "dhilajfakja");`


### SET

```sql
SET enable_seqscan TO 'off'
```

- デフォルトでSESSIONとなる。SESSIONは現在のセッション（コネクション）単位。
- LOCALにすると、現在のトランザクション単位となる。

```sql
SET LOCAL enable_seqscan TO 'off'
```

- (参考) [PostgreSQL: SET](https://www.postgresql.jp/docs/14/sql-set.html)


### 型の変更

- (参考) [PostgreSQLのカラムの型変更](https://www.javadrive.jp/postgresql/table/index16.html)
- 既に入っている値によっては型を変更できない場合がある。その場合はUSINGを使って変換方法を指定する。

例1（varcharからnumericへ）:

```sql
ALTER TABLE spots ALTER latitude TYPE numeric(15,12) USING latitude::numeric(15,12);
```

例2（smallintからbooleanへ）:

```sql
ALTER TABLE users ALTER is_active TYPE boolean USING is_active::INT::BOOLEAN;
```

ポイントとして、`is_active::BOOLEAN` だと直接エラーになるため、INTを経由していること。

- (参考) [Casting smallint to boolean in PostgreSQL - Stack Overflow](https://stackoverflow.com/questions/31343809/casting-smallint-to-boolean-in-postgresql)


### offset（開始位置）

以下のどちらか。

```sql
SELECT カラム名, ... FROM テーブル名 LIMIT 行数 OFFSET 開始位置;
SELECT カラム名, ... FROM テーブル名 LIMIT 開始位置, 行数;
```


### トランザクション

```sql
begin;
select * from users where id = '10101010-1010-1010-1010-101010101010';
commit;
# rollback;
```
