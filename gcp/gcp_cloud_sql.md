---
title: "Cloud SQL - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > Cloud SQL

* インスタンス作成のコマンド例は[サービス別のコマンド例](./gcp_cli_examples.md)、確約利用割引(CUD)は[料金・無料枠・CUD](./gcp_billing_cud.md)を参照。

## データベースの選定(IMO)

### Cloud SQL
* 料金は安くないので、少なくともテスト期間は一番小さい構成を使うのが良さそう。
* CUD(確約利用割引)を利用すると安くできる。
* Cloud Runを使うのであれば、外部のマネージドDBだとリモート通信で遅くなる可能性があるため、Cloud SQLのほうが良さそう(?)。
    * Cloud SQLならUnixドメインソケット経由で接続できる。ただし、リージョンが異なると遅くなると思われる(?)。

### Cloud Spanner
* オートスケールする。AWSでいうAuroraに近い位置づけ(?)。
* ただしMySQLやPostgreSQLと完全な互換ではなく、既存のアプリケーションをそのまま載せることはあまり推奨されていない。
* そういう意味では、AWSのAuroraのほうが使い勝手は良さそう(IMO)。

### AlloyDB
* 2022年5月に発表された、PostgreSQL互換のデータベースサービス。
* Cloud Spannerのようにクラウド技術を活かして拡張性や性能を高めつつ、PostgreSQLとの完全互換をうたっている。
* Cloud SpannerよりもPostgreSQLとの互換性が高く、既存のPostgreSQLアプリケーションをコードの変更なしに移行可能とされる。
* 将来的にデータベースの規模が大きくなったら、Cloud SQL -> AlloyDBに移行する、という選択肢もありそう(IMO)。

### RDBMS選択フローチャート
* (参考) https://dev.classmethod.jp/articles/google-cloud-rdbms-devio2024/


## Cloud SQL Auth Proxy
* https://cloud.google.com/sql/docs/postgres/sql-proxy
* 承認済みネットワークやSSLの構成を必要とせず、安全にインスタンスへアクセスできるCloud SQLコネクタ。
* IAM権限を使用して、Cloud SQLインスタンスに接続できるユーザーと対象を制御する。
* TLS 1.3と256ビットAES暗号を使用して、データベースとの間で送受信されるトラフィックを自動的に暗号化する。
* クライアント側の起動などはCloud Runを使う場合はマネージドで処理されるため、そこまで意識する必要はない。手動でCloud SQL Auth Proxyを使って接続する際は考慮事項が増える。


## Cloud SQL Auth Proxyを使用してローカルのDBクライアントから接続する
* https://cloud.google.com/sql/docs/mysql/connect-auth-proxy#unix-sockets
* 最新バージョンの確認: https://github.com/GoogleCloudPlatform/cloud-sql-proxy/releases

```sh
# ADCの認証をしていない場合
gcloud auth application-default login

# プロキシのバイナリを取得(バージョン・プラットフォームは適宜置き換える)
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.14.0/cloud-sql-proxy.darwin.arm64
chmod +x cloud-sql-proxy

# Unixソケット用のディレクトリを用意
mkdir cs
chmod 777 cs

gcloud --project [PROJECT_ID] sql instances list
gcloud --project [PROJECT_ID] sql instances describe [INSTANCE_NAME]

./cloud-sql-proxy [PROJECT_ID]:[REGION]:[INSTANCE_NAME] --unix-socket ./cs
```

* DBクライアント(TablePlus等)のhost / socketの欄に、作成したソケットのパスを入力する。
    ```
    /path/to/proxy/cs/[PROJECT_ID]:[REGION]:[INSTANCE_NAME]
    ```
* 注意: Unixソケットのパスは103バイトの制限があるため、プロジェクトIDやインスタンス名が長いと制限に引っかかる(IME)。作業ディレクトリを浅い場所に置くとよい。


## インスタンスの設定・デフォルト値
* https://cloud.google.com/sql/docs/postgres/instance-settings#settings-2ndgen
* インスタンス作成後に変更できるかどうかは、ドキュメントの「作成後の変更の可否」の列に記載されている。

### リージョン
* インスタンスが配置されるGoogle Cloudリージョン。
* インスタンス作成時にのみ設定可能。
* パフォーマンスを向上させるには、そのデータを必要とするサービスに近い場所でデータを保存する。

### ゾーン
> インスタンスが配置される Google Cloud のゾーン。Compute Engine インスタンスから接続する場合は、Compute Engine インスタンスが存在するゾーンを選択します。それ以外の場合は、デフォルトのゾーンをそのまま使用します。

### エディション
* デフォルトはEnterpriseエディション。アップグレードするとEnterprise Plusとなる。
* 価格はPlusが1.3倍ほどになる。
* PostgreSQLのバージョンによってはデフォルトがEnterprise Plusになる場合がある。
* (参考) https://zenn.dev/hi_ka_ru/articles/cloudsql-20240602

### SSD / HDD
* SSDがデフォルト。
* 途中でインスタンスのHDD / SSDを他方に変更することはできない。

### バックアップ / ポイントインタイムリカバリ
* https://cloud.google.com/sql/docs/postgres/instance-settings#backups-and-binary-logging-2ndgen
    > これらのオプションにより、自動バックアップが実行されるかどうか、および write-ahead log 書き込みを有効にするかどうかが決まります。どちらのオプションでも若干のパフォーマンス コストと追加のストレージが発生しますが、レプリカとクローンの作成およびポイントインタイム リカバリには必要です。このオプションを選択すると、自動バックアップを実行する時間帯も選択できます。
    > 自動バックアップは、選択した時間帯に毎日行われます。7 日後に、最も古いバックアップが削除されます。
* デフォルトで有効となっている。
* 注意: バックアップが有効だとしても、バックアップのスケジュールを設定しないとバックアップ自体は行われない。
    * ただし`gcloud sql instances create --help`では下記のように書かれており、オプションの説明が正確ではないように見える(?)。
        ```
        --backup
        Enables daily backup. Enabled by default, use --no-backup to disable.
        ```
* 料金
    * (参考) https://blog.g-gen.co.jp/entry/cloud-sql-explained
* TODO: 以下は未整理
    * バックアップ: https://cloud.google.com/sql/docs/postgres/backup-recovery/backups
    * オンデマンドバックアップの作成 / バックアップスケジュールの設定: https://cloud.google.com/sql/docs/postgres/backup-recovery/backing-up
    * ポイントインタイムリカバリ: https://cloud.google.com/sql/docs/postgres/backup-recovery/pitr

### マシンタイプ
* Cloud SQL Enterpriseエディションのインスタンスの形式
    * `db-custom-(vCPU数)-(メモリ量)`
    * vCPUは1、または2〜96の間の偶数にする必要がある。
    * メモリは次の条件を満たす必要がある。
        * vCPUあたり0.9〜6.5GB
        * 256MBの倍数
        * 3.75GB(3,840MB)以上
    * 過去の形式
        * `db-n1-standard-1`のような指定はレガシーな形式であり、現在は個別のマシンタイプという概念は無い。
        * `db-n1-standard-1`は上記の形式では`db-custom-1-3840`に該当する(1vCPU、メモリ3840MB)。
    * コンソールの場合はCPUやメモリ量を選択すればよい(目安として「軽量」「標準」「ハイメモリ」という区分けもある)。CLIの場合は`--memory`や`--cpu`で指定する。
* 共有コアCPUを使用するマシンタイプ
    * `db-f1-micro`と`db-g1-small`。
    * これらを使う場合は`--tier`で指定する必要がある(この場合は`--memory`、`--cpu`は不要)。
    * Cloud SQL SLAには含まれない。
    * 開発用インスタンスにのみ使用することが想定されている。
* https://cloud.google.com/sql/docs/mysql/instance-settings

### ストレージ容量
* デフォルトは10GB(?)。
* 後から変更できるが、増加のみ。
* また、自動増量がデフォルトで「オン」となっている点に注意。

### ロケーションオプション
* バックアップを複数のリージョンに保存するか、単一のリージョンに保存するか。
    * 高可用性の話とは別。
* デフォルトはマルチリージョン。

### 可用性: シングルゾーン
* インスタンスとバックアップを1つのゾーンに配置する。
    * この設定と「ロケーションオプション」がマルチリージョンの場合どうなるのかは未確認(?)。
* このオプションを選択すると、停止時にフェイルオーバーは発生しない。
* テストと開発の目的でのみ使用することが推奨されている。
* デフォルトはオン。

### 高可用性(リージョン)
* デフォルトはオフ。
* インスタンスはリージョン内の別のゾーンにフェイルオーバーする。
* メンテナンス時のメリットもある。
    > 高可用性（HA）とプライベート IP 接続を備えた Cloud SQL Enterprise Plus エディションのプライマリ インスタンスの場合、通常、メンテナンス ダウンタイムは 1 秒未満
    * ただし、リードレプリカについてはメンテナンスの時間枠の設定がサポートされていない(?)。


## 料金
* https://cloud.google.com/sql/pricing
* 料金計算ツール: https://cloud.google.com/products/calculator

### 試算のサンプル(執筆時点の東京リージョンでの試算)
* 構成: 1vCPU、3.75GiB、ディスク10GB
    * 料金計算ツールでの見積もりは月額66ドル程度だった。
* 内訳(単価は変動するため、必ず最新の料金ページを確認すること)
    * CPU: $39.201 per vCPU
    * メモリ: $6.643 per GB (3.75GBで24.91ドル程度)
    * ディスク(SSD): $0.221 per GB/month (10GBで2.21ドル、30GBで6.63ドル)
    * ネットワーク料金
        * Ingressは無料。
        * Egressは有料だが、通常の用途ではほぼかからないだろう(IMO)。
        * `IPv4 addresses: $0.015 per hour while idle` という項目があるが、これはインスタンスを停止していなければかからない、ということだと思われる(?)。
* 料金表から手計算した結果と料金計算ツールの結果に若干のズレがあり、原因はディスクの計算(730時間を1ヶ月として扱うかどうか)と考えられる。


## Cloud SQL Studio
* コンソール画面から直接DBに対してSQLを実行できる機能。
* (参考) https://zenn.dev/hi_ka_ru/articles/cloudsql-20240602


## パスワードポリシーの設定
* TODO: 未整理。
