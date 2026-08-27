---
title: "コマンド・操作 - RDB(MySQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > コマンド・操作（MySQL）


## コマンド・操作（MySQL）

### 接続

```
mysql -u root -p
```

ログイン。`-p` でパスワード入力、`-D データベース名` でデータベース指定。


### データベース・テーブル

```sql
show databases;
USE データベース名;
select database();  -- 現在のデータベースを確認
show tables;
desc テーブル名;  -- データベース名でもOK
```

データベース作成:

```sql
create database if not exists データベース名 default character set utf8;
grant all on データベース名.* to ユーザー名@localhost identified by 'パスワード';  -- 作成したDBに権限のあるユーザー・パスワードを設定
```


### ユーザー・権限

ユーザー一覧・変更・削除:

```sql
SELECT user, host FROM mysql.user;  -- ユーザー一覧を見る
SET PASSWORD FOR 'ユーザー名'@'localhost' = password('パスワード');  -- パスワード変更
drop user 'ユーザー名'@'%';  -- もしくは drop user 'ユーザー名'@'localhost';  -- ユーザー削除
```

ユーザーの追加・SELECTのみの権限付与:

```sql
CREATE USER 'select_user'@'%' IDENTIFIED BY 'パスワード';
SHOW GRANTS FOR 'select_user'@'%';
grant select on myproject.* to 'select_user'@'%';
```

`%` はどこからでもアクセスできるユーザーという意味(?)。`localhost` だとlocalhostからしかアクセスできない(?)。

パスワードのアップデート:

```sql
set password for 'select_user'@'%' = '新しいパスワード';
```

`FLUSH PRIVILEGES` が必要なケース: 単純なユーザー追加程度であれば不要。

- (参考) [FLUSH PRIVILEGESが必要なケース](https://zatoima.github.io/mysql-flush-privileges.html)


### エクスポート・インポート

```sql
mysqldump --add-drop-table -h localhost -u ユーザー名 -p データベース名 > backup.sql  -- エクスポート
mysql -u root -p データベース名 < backup.sql  -- インポート（あらかじめcreateしておくこと）
mysqldump -u root -p -x --all-databases > dump.sql  -- 全てエクスポート
mysql -u root -p < dump.sql  -- 全てインポート
```


### データベースのリネーム

`RENAME DATABASE` は危険なため削除されている。代わりに、新しいDBをCREATEして、そこに旧DBからデータをdumpし、dumpしたデータを流し込む。最後に元のDBをdropする。

```sql
-- バックアップ作成
mysqldump --add-drop-table -h localhost -u ユーザー名 -p 旧データベース名 > バックアップファイル名.sql
mysql -u root -p
-- 新しいDB作成
create database if not exists 新しいデータベース名 default character set utf8;
mysql -u root -p 新しいデータベース名 < バックアップファイル名.sql
-- バックアップファイルは削除
```


### データベースのコピー

一発でコピーできるコマンドはないので、新しいDBをCREATEして、コピー元からdumpし、dumpしたデータをコピー先に流し込む。

- (参考) [MySQLデータベースのコピー方法](https://weblabo.oscasierra.net/mysql-database-copy/)


### 全て削除（初期化）

```
yum remove mariadb mariadb-server
rm -rf /var/lib/mysql
```


### コネクション確認

```sql
SHOW STATUS LIKE '%connect%';
```

- `Connections`: 起動してからの累積接続数
- `Max_used_connections`: 起動してからこれまでの最大同時接続数
- `Threads_connected`: 現在の接続数

```
+-----------------------------------------------+-------+
| Variable_name                                 | Value |
+-----------------------------------------------+-------+
| Aborted_connects                              | 4     |
| Connection_errors_accept                      | 0     |
| Connection_errors_internal                    | 0     |
| Connection_errors_max_connections             | 0     |
| Connection_errors_peer_address                | 0     |
| Connection_errors_select                      | 0     |
| Connection_errors_tcpwrap                     | 0     |
| Connections                                   | 2234  |
| Max_used_connections                          | 2     |
| Performance_schema_session_connect_attrs_lost | 0     |
| Slave_connections                             | 0     |
| Slaves_connected                              | 0     |
| Ssl_client_connects                           | 0     |
| Ssl_connect_renegotiates                      | 0     |
| Ssl_finished_connects                         | 0     |
| Threads_connected                             | 1     |
| wsrep_connected                               | OFF   |
+-----------------------------------------------+-------+
```


### 状態の確認

プロセスの確認:

```sql
SHOW PROCESSLIST;
```

コネクション数:

```sql
show status like 'Threads_connected';
SHOW GLOBAL VARIABLES like 'max_connections';
```
