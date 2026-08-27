---
title: "設定・ログ・ディレクトリ - RDB(MySQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > 設定・ログ・ディレクトリ（MySQL）


## 設定・ログ・ディレクトリ（MySQL）

### 設定ファイル

`my.cnf` の場所確認:

```
locate my.cnf
/etc/my.cnf
```

設定変更:

```
vi /etc/my.cnf
```


### ログの確認

```
cat /var/log/mariadb/mariadb.log
```


### InnoDB

InnoDBだと途中でSQLを中断すると、ロールバックされる。

- (参考) [What happens when an UPDATE command is interrupted - Stack Overflow](https://stackoverflow.com/questions/18168712/mysql-what-happens-when-an-update-command-is-interrupted)

MySQLサーバーのデフォルトストレージエンジンは設定ファイルで規定され、記述がない場合はバージョン次第(?)。`show engines;` で確認することができる。下記の例だとデフォルトはInnoDBになっている。

```
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| myproject           |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+

mysql> use information_schema;
Database changed
mysql> select table_schema, table_name, engine from tables;
+--------------------+------------------------------------------------------+--------------------+
| TABLE_SCHEMA       | TABLE_NAME                                           | ENGINE             |
+--------------------+------------------------------------------------------+--------------------+
| myproject           | admin_settings                                       | InnoDB             |
```

#### MyISAM

MyISAMのデータベースではトランザクション機能がない。後発で生まれたInnoDBのデータベースにはトランザクション機能がある。(IMO) 普通はInnoDBを使うので、MyISAMを使うことはほぼないだろう。


### ディレクトリ・ログ・設定ファイル

#### 初期設定

- デフォルト設定ファイルは `/etc/my.cnf`。ただし、中身は空で `/etc/my.cnf.d/` 配下の設定ファイルをインクルードするようになっている。
- `/etc/my.cnf.d/` ディレクトリ内にあるファイルの中身も、同じくほとんど設定が入っていない場合が多い(?)。
- 結局、MySQLのサンプル設定が入っているディレクトリ `/usr/share/mysql/` から設定ファイルを持ってくる必要がある。
- `/usr/share/mysql/` 内には、データベースの規模に応じて使えるように各種設定ファイルが用意されている。

#### データが格納されるところ

- `/var/lib/mysql` ディレクトリ（データ本体が入るところ）。
- MariaDBをインストールするとデフォルトで `/var/lib/mysql` が作成される。

#### データベースのディレクトリ

- データベースを作成すると、データベース名のディレクトリが `/var/lib/mysql` に作成される。
- 最初は `db.opt` しかないが、テーブルを作成すると以下のファイルが作成される。
    - `.frm` ファイル: フィールド定義などテーブル情報を記録するファイル。
    - `.ibd` ファイル: テーブルごとにディレクトリとインデックスを記録するファイル（テーブルスペースではなく、ibdファイルに記録する）。

#### トランザクションログ

MySQLのトランザクションログには、InnoDBログとバイナリログがある。InnoDBログとバイナリログの違いはやや理解が曖昧(TODO)。InnoDBログはインスタンス障害時に次回起動時に自動で復旧させるためのもの、バイナリログは自分で復旧させる際に使うもの、という理解をしている(?)。

- (参考) [MySQLのトランザクションログについて](https://yohei-a.hatenablog.jp/entry/20180303/1520079846)
- データベースに適用された更新処理を記録しておくファイル。プロセス障害からの復旧、ファイル消失・破損からの復旧に使用される。

InnoDBログファイル:

- 更新処理の高速化に使われる（WAL: Write Ahead Log）。また、MySQLがクラッシュした際のリカバリ処理にも利用される。
- クラッシュリカバリ: インスタンスが異常終了した際に、次回起動時にデータベースの整合性を回復する。InnoDBログはロールフォワードに使われる（ロールバックにはUNDOログが使われる）。
- (IME) Oracle DatabaseのREDOログによるロールフォワード、UNDO表領域のロールバックセグメントでロールバックする話と近い構造だと理解している(?)。

バイナリログファイル:

- レプリケーションのMaster → Slave同期処理に利用される。
- バックアップデータ復元後のロールフォワード処理にも利用される（メディアリカバリ）。バックアップからリストアし、そこからバックアップが実行された後に記録されたバイナリログ内のイベントが再実行（ロールフォワード）される。
- (IME) Oracle Databaseで言うと、インポートがリストアに、バイナリログでのロールフォワードがアーカイブログとREDOログを使ったロールフォワードに当たると理解している(?)。
- ただし、出力することで若干の性能低下がある。

#### テーブルスペースとログファイルの具体的な構成

`/var/lib/mysql` に以下が入っている。

- InnoDB共有テーブルスペース（`ibdata*`）: 全データを管理する共有テーブルスペース。
- InnoDBログファイル（`ib_logfile0`, `ib_logfile1`）
    - (参考) [InnoDBログファイルについて](https://yohei-a.hatenablog.jp/entry/20180303/1520079846)
    - `ib_logfile0`, `1` はログファイル（OracleではREDOログに相当）。
    - InnoDBログファイルは1つのデータベース領域に少なくとも2つ作成される。ログそのものには障害に備えるミラーリング機能はない。
    - InnoDBのデータはテーブルスペースとログを合わせて完全な情報になる。データは直接テーブルスペースに更新されるのではなく、一旦ログファイルに更新内容が書き込まれ、その後にテーブルスペースに反映される。テーブルスペースの更新はコストが高いためすぐに反映するのではなく、ログファイルに書いておくことで書き込み性能を上げている(?)。そのためログだと思って削除してしまうと問題が起きるので注意。
- バイナリログファイル（`bin-log.000001…`）
    - `--slow-query-log` を付けてmysqldを起動すると有効になる（初期では存在しない）。`my.cnf` に `log-bin` を追記しても有効化できる(?)。
    - すべてのストレージエンジンに対するテーブルの作成・削除、データの挿入・更新・削除といった変更操作を記録した更新ログファイル。記録されるのは更新系のクエリのみ。ログファイル名は `my.cnf` で定義できる。
    - ログがスイッチすると拡張子の数値が繰り上がる。

#### その他

- スロークエリログ: `--log-bin` を付けてmysqldを起動すると有効になる(?)。
- エラーログ、一般クエリログ: 詳細未調査(TODO)。
- 初期の容量管理について
    - `ibdata*` はデータベースの中身が増えていくと増大していく。データベースを定期的に削除していればそれに伴って減るはず(?)。定期的にデータを削除する設計にすればよい。
    - ログファイル（`ib_logfile*`）は、1つのDBあたり2つ程度しか作成されない(?)。
    - バイナリログやスロークエリログはデフォルトだと作成されない。バイナリログは放っておくとどんどん増え続ける。メンテナンスするには `my.cnf` で `expire-logs-days=7` などと指定すればよい（古い情報の可能性がある(?)）。
    - ログファイルのサイズについて、チェックポイントでテーブルスペースへ反映されるログファイルにはそれなりにデータがたまっていく。適切なサイズの目安は精査していない(TODO)。`innodb_log_file_size` で調整できるが、変更するとログファイルを削除しないとエラーになるケースがある(?)。

参考:

- (参考) [CentOS 7 MariaDB install and mysql secure installation](http://server.etutsplus.com/centos-7-mariadb-install-and-mysql-secure-installation/)
- (参考) [MariaDB Logs Overview](https://mariadb.com/kb/ja/overview-of-mariadb-logs/)
- (参考) [MySQLログについて](http://d.hatena.ne.jp/graySpace/20140911/1410450256)

#### ログの削除

- (参考) [ログ削除に関するメモ](https://www.skyarch.net/blog/?p=1096)
- (参考) [ログ削除に関するメモ - Qiita](https://qiita.com/Brutus/items/5244685a2b5e6b7d2a54)


### クエリキャッシュ

確認:

```sql
SHOW VARIABLES LIKE '%query_cache%';
```

以下は、クエリキャッシュが無効になっている例。キャッシュ領域も小さい。

```
+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| have_query_cache             | YES     | キャッシュクエリを使用可能か
| query_cache_limit            | 1048576 | クエリキャッシュ最大サイズ(1MB)
| query_cache_min_res_unit     | 4096    |
| query_cache_size             | 1048576 | クエリキャッシュ領域(1MB)
| query_cache_strip_comments   | OFF     |
| query_cache_type             | OFF     | クエリをキャッシュするか
| query_cache_wlock_invalidate | OFF     | 書込みロック時にロックしたテーブルに関するクエリキャッシュを無効にするか
+------------------------------+---------+
```

※ 最後の項目は、更新中にキャッシュから結果を返したら困る場合に設定する。

```sql
SHOW STATUS LIKE 'Qcache%';  -- クエリキャッシュの状態確認
```

`Qcache_free_blocks` が1でないときは断片化しているため、`FLUSH QUERY CACHE` でキャッシュのデフラグも検討したほうがよい(?)。以下の例では、キャッシュの設定がOFFになっていることがわかる。

```
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| Qcache_free_blocks      | 1       | 空きメモリーブロック数。起動時は1でOK。
| Qcache_free_memory      | 1031320 | クエリキャッシュ用の空きメモリーの量
| Qcache_hits             | 0       | クエリキャッシュヒットの数
| Qcache_inserts          | 0       | クエリキャッシュに追加されるクエリの数
| Qcache_lowmem_prunes    | 0       | メモリーが少ないためクエリキャッシュから削除されたクエリの数
| Qcache_not_cached       | 0       | 非キャッシュクエリの数
| Qcache_queries_in_cache | 0       | クエリキャッシュ内に登録されたクエリの数
| Qcache_total_blocks     | 1       | クエリキャッシュ内のブロックの合計数
+-------------------------+---------+
```

`my.cnf` への記載例（`[mysqld]` セクションの下に書く。一番下に書くとエラーになる点に注意）:

```
[mysqld]
# #################
# query cache
# #################
query_cache_limit=16M
query_cache_size=128M
query_cache_type=1
```

再起動して確認する:

```
systemctl restart mariadb.service
```

(IME) WEBサイトにアクセスして `SHOW STATUS LIKE 'Qcache%';` で確認したところ、memcachedが効いていたため `Qcache_hits` 自体は変わらなかったが、`Qcache_inserts` が増えていたのでクエリキャッシュ自体は機能していることが確認できた。

```
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| Qcache_free_blocks      | 1         |
| Qcache_free_memory      | 134180216 |
| Qcache_hits             | 0         |
| Qcache_inserts          | 12        |
| Qcache_lowmem_prunes    | 0         |
| Qcache_not_cached       | 1         |
| Qcache_queries_in_cache | 12        |
| Qcache_total_blocks     | 27        |
+-------------------------+-----------+
```
