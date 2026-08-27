---
title: "復旧・復元 - RDB(MySQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > 復旧・復元（MySQL）


## 復旧・復元（MySQL）

### mysqldumpによる単純な復旧方法

あまりよくないが、単純にdumpファイルをインポートする方法。トランザクションが極端に少ないテーブルの場合などに使う(?)。事前にバックアップを取っておき、問題があればバックアップで復旧する、という使い方が想定される。

バックアップはデータベース全体ではなく、テーブル単位で取っておくのがよさそう(IMO)。トランザクションが多いテーブルまで復元してしまうのはリスクが高いため。

エクスポート:

```
mysqldump --single-transaction -u user DB名 テーブル名A テーブル名B > dump.sql
```

- `--single-transaction`: InnoDBのデータベースをdumpする際はとりあえずつけておいたほうがよい(IMO)。
    - (参考) [mysqldumpで--single-transactionをつけるべき理由](https://masyus.work/articles/if-using-mysqldump-add-single-transaction-skip-lock-table/)
    - `mysqldump` では `--opt` がデフォルトで有効になっている。`--opt` は複数のオプションを一括で有効にするオプションで、`--lock-tables` などが含まれる。
    - `--lock-tables` は、dumpしている最中のテーブルに`READ LOCK`をかける。ロックには`READ`と`WRITE`の2種類がある。
    - `--lock-tables` によって、mysqldumpしているセッション以外のセッションでinsert/update/deleteができなくなる（CRUDのうちRead以外の処理が止まる。ちなみに`WRITE LOCK`はReadもできなくなる）。
    - そもそもなぜ `--lock-tables` がデフォルトで有効になっているかというと、InnoDB以前の、トランザクション機能がなかったMyISAMのためと思われる(?)。したがって、トランザクション機能が使えるInnoDBの場合は `--single-transaction` を使えば、わざわざロックをする必要はない。
    - `--single-transaction` をつけると、REPEATABLE READが有効になる。つまり一貫性読み取り（ある瞬間のスナップショットデータに対しての読み込み）が可能になり、わざわざテーブルをREAD LOCKさせることなくdumpできる。その性質上、dump中にinsertされたデータもdumpデータに含めたい場合はこの限りではないが、そもそもそのようなケースはほぼ稀だと思われる(IMO)。
    - SQLにはinsert以外に、drop tableやcreate tableが含まれる。

インポート:

```
mysql -u root -p < dump.sql
```

- (参考) [mysqldumpの基本 - Qiita](https://qiita.com/katsukii/items/c7709fc501c1eb11603f)


### ロールフォワードによる復旧

- (参考) [ロールフォワードによる復旧手順](https://sys-guard.com/post-12123/)

前提:

- 定期的にバックアップされていること。
- バイナリログがONになっていること。

定期的バックアップの例:

```
mysqldump mysqltest_db --single-transaction --master-data=2 --flush-logs > /root/mysqltest_db.`date "+%Y%m%d_%H%M%S"`.sql
```

障害発生後の手順:

1. 障害が発生したらメンテナンスモードにする（これ以上トランザクションを発生させないため）。
2. 現時点のダンプデータを取得する。

    ```
    mysqldump mysqltest_db --single-transaction --master-data=2 --flush-logs > /root/mysqltest_db.rollback_before_`date "+%Y%m%d_%H%M%S"`.sql
    ```

3. 定期バックアップの最新のチェックポイントのマスターログとポジションを確認する。

    ```
    head -n 100 /root/mysqltest_db.20161124_145637.sql
    ```

    ```
    ・・・・
    CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000015', MASTER_LOG_POS=120;
    ・・・・・
    ```

    上記から、チェックポイント時点では `mysql-bin.000015` の120のポジションまで進んでいたことがわかる。

4. `mysql-bin.000015` から問題のクエリのポジションを探す。
5. 問題のクエリの実行前のポジションまで実行するクエリを作成する。

    ```
    mysqlbinlog --database="mysqltest_db" --start-position=120 --stop-position=1085 /var/lib/mysql/mysql-bin.000015 > /root/recovery_1.sql
    ```

6. 問題のクエリの実行後のポジションから最後のクエリまで実行するクエリを作成する（例省略）。
7. 最新のチェックポイントをリストアする。
8. 復旧クエリを実行し、ロールフォワードする。

    ```
    mysql -u root mysqltest_db < recovery_1.sql
    mysql -u root mysqltest_db < recovery_2.sql
    ```

9. メンテナンスモードを終了する。
