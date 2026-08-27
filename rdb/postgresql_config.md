---
title: "設定・環境 - RDB(PostgreSQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > 設定・環境（PostgreSQL）


## 設定・環境（PostgreSQL）

### ドキュメント

- (参考) [PostgreSQL 14 文書](https://www.postgresql.jp/document/14/html/)


### 設定確認

```sql
show all
select name, setting, unit from pg_settings
select name, setting, unit from pg_settings where name = 'lock_timeout'
```


### タイムゾーン

- タイムゾーンの変更
    - (参考) [ALTER TIMEZONE](https://bitto.jp/alter-timezone/)
    - ただし、一度セッションを切らないと変わらない。立ち上げているサーバーについても、同じセッションで接続しているとみなされる(?)ため、一度サーバーをリスタートしないと変わらない。
- タイムゾーンの確認

```sql
show timezone;
```


### lock_timeout

- (参考) [PostgreSQL: クライアント接続のデフォルト設定](https://www.postgresql.jp/document/10/html/runtime-config-client.html)
- `lock_timeout` のデフォルトは0であり、無効になっている。
- ドキュメント上は、`postgresql.conf` に設定するとすべてに影響を与えるため非推奨とされている。
- (IMO) 気持ちとしてはロックはすべて待たせたくないので1msに設定したいところだが、あまり一般的ではなさそう。


### statement_timeout

- (参考) [PostgreSQL: クライアント接続のデフォルト設定](https://www.postgresql.jp/document/14/html/runtime-config-client.html)
- ステートメントごとの実行タイムアウト。
- デフォルトは0で、その場合は機能自体が無効。
- 設定したほうがよいかどうかは要検討(TODO)。


### idle_in_transaction_session_timeout, idle_session_timeout

- (参考) [PostgreSQL: クライアント接続のデフォルト設定](https://www.postgresql.jp/document/14/html/runtime-config-client.html)
- デフォルトは0で、その場合は機能自体が無効。
- `idle_session_timeout` は普通は特に設定する必要はない(?)。詳細は要検討(TODO)。


### クライアント・サーバモデル

- (参考) [PostgreSQL: アーキテクチャの基礎](https://www.postgresql.jp/document/14/html/tutorial-arch.html)
- PostgreSQLはクライアント/サーバモデル。
    - PostgreSQLのセッションは、以下の協調動作するプロセス（プログラム）から構成される。
        - サーバプロセス: データベースサーバプログラムはpostgresと呼ばれている。
        - ユーザのデータベース操作を行うクライアント（フロントエンド）アプリケーション。
    - クライアントとサーバはTCP/IPネットワーク接続経由で通信を行う。
- PostgreSQLサーバはクライアントから複数の同時接続を取り扱うことができる。
    - スーパーバイザサーバプロセスは常に稼働してクライアントからの接続を待ち続け、一方でクライアントからの接続と関連するサーバプロセスの起動が行われる。
        - サーバは接続ごとに新しいプロセスを開始（fork）する。
        - クライアントと新しいサーバプロセスは、元のpostgresプロセスによる干渉がない状態で通信を行う。


### 同時接続

```sql
select * from pg_stat_activity
```

- セッション＝コネクションのことと理解している(?)。
- 最大同時接続数（デフォルトは100）

```sql
SHOW max_connections;
```


### max_connections

増やしすぎるとメモリ不足によるスワップが起きる可能性もある。

- (参考) [PostgreSQL: max_connectionsに関するトラブルシューティング](https://lets.postgresql.jp/documents/tutorial/troubleshoot/1)
