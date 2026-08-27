---
title: "インデックス - RDB(PostgreSQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > インデックス（PostgreSQL）


## インデックス（PostgreSQL）

### Seq ScanとIndex Scan

データ件数が少なく、Seq Scanの方が効率的と判断された場合は、Index Scanが可能であってもオプティマイザによりSeq Scanが選択される。

以下のようにoffにすることで、Index Scanを優先させることが可能（ただしSeq Scanしか使えない場合はSeq Scanになる）。

```sql
SET enable_seqscan TO 'off'
select name, setting, unit from pg_settings where name = 'enable_seqscan'
```

戻す場合:

```sql
SET enable_seqscan TO 'on'
```


### インデックスの作成例

```sql
CREATE INDEX index__user_datas__user_id ON "user_datas" (user_id);
```

削除:

```sql
DROP INDEX index__user_datas__user_id;
```


### 複合インデックス

単一のカラムだけしか使わない場合、そこまで速度の違いはなさそう(?)。

- (参考) [複合インデックスについて - Zenn](https://zenn.dev/jnuank/articles/0da8d4755e69fea30bab)


### JOIN

- JOINでは、インデックスが貼られていない場合はスキャンの計算量がN^2になる。
- インデックスを使うと計算量はNlogNくらいになる(?)。
- WHEREではなくON句を使ったり、副問い合わせを使ったりする工夫でNを減らしたり、JOIN自体を減らす。
- (参考) [JOINとインデックスについて](http://www.code-magagine.com/?p=14618)


### 主キー(PRIMARY)・一意キー(UNIQUE)とユニークインデックス

- (参考) [PostgreSQL: ユニークインデックス](https://www.postgresql.jp/document/14/html/indexes-unique.html)
- 一意キー(UNIQUE)はテーブルに付与する制約。主キー(PRIMARY)も同様。
- ただし、
    - 主キー(PRIMARY)は1つのテーブルに対し1つしか作成できない。
    - 一意キー(UNIQUE)は1つのテーブルにいくつでも設定することが可能。
- 「ユニークインデックス」はテーブルの制約ではなくオブジェクト。
    - 通常のインデックスと違う点は `is_unique` が `true` となっているところ(?)。
    - ユニークにしたい場合は、ユニークインデックスを直接作成する（`CREATE UNIQUE INDEX〜`）のではなく、主キーや一意キーを作る（`ALTER TABLE〜`）のがよさそう(IMO)。

#### 一意キー(UNIQUE)の作成

下記を実行すると、ユニーク制約とともに `user_status_user_id_key` という名前でユニークインデックスのオブジェクトが作成される。

```sql
CREATE TABLE "user_status" (
    "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
    "user_id" uuid NOT NULL UNIQUE
);
```

下記でも同じ結果となる。

```sql
CREATE TABLE "user_status" (
    "id" uuid NOT NULL DEFAULT uuid_generate_v4(),
    "user_id" uuid NOT NULL
);
ALTER TABLE "user_status" ADD CONSTRAINT "user_status_user_id_key" UNIQUE("user_id");
```

#### 一意キー(UNIQUE)の削除

ユニークインデックスを直接削除しようとするとエラーとなる。

```sql
DROP INDEX user_status_user_id_key;
/*
DROP INDEX user_status_user_id_key; (details: ERROR: cannot drop index user_status_user_id_key because constraint user_status_user_id_key on table user_statuses requires it (SQLSTATE 2BP01))
*/
```

下記のように制約を削除することで、制約とユニークインデックスの両方が削除される。

```sql
ALTER TABLE "user_statuses" DROP CONSTRAINT "user_status_user_id_key";
```

#### 主キー

- (参考) [主キーについて - Qiita](https://qiita.com/jiyu58546526/items/640435de0bae90e30e35)
