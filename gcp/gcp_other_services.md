---
title: "その他のサービス(App Engine・BigQuery) - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > その他のサービス

* このページは単独のページにするほどの分量が無いサービスのメモをまとめたもの。TODO: 内容が増えたら分割する。

## App Engine (GAE)

### 注意点
* App Engineを有効化する際にはリージョンを選ぶが、一度選ぶと変更できない。
* 使ってみたいだけであれば、サンドボックス用のプロジェクトなど、消してしまってよいプロジェクトで試すこと。
* (参考) https://attsun1031.github.io/blog/gcp-three-things

### GAEとGCEの違い
* GAE ≒ Heroku (PaaS)
* GCE ≒ EC2 (IaaS)

### メリット
* サーバー管理が不要(SSHログインして作業する、といったことが必要ない)。
* Dockerと比較すると、そもそもコンテナの構成管理が必要ない。
* オートスケールするため、アプリの開発に集中できる。
* 使える言語には制限がある。
* IAPという仕組みでほぼ無料でアクセス制御ができる。
    * [セキュリティ(IAP・Cloud Armor・IDトークン)](./gcp_security.md)を参照。
    * (参考) https://zenn.dev/catnose99/articles/5e9547a5c207e3

### デプロイ方法
* `gcloud app create --project=[PROJECT_ID]`
    * `--project`を省略すると、現在のコンフィグのプロジェクトに対して作成される。
* プロジェクト内で`gcloud app deploy`を実行する。設定ファイルを指定して`gcloud app deploy app.standard.yaml`のようにデプロイすることも可能(?)。
* `gcloud app browse`でブラウザから開ける(?)。
* https://cloud.google.com/appengine/docs/standard/nodejs/quickstart

### (IMO) 現在の位置づけ
* 現在のGoogle Cloudでは、サーバーレスな用途は基本的にCloud Runが第一候補になる。新規に構築するのであればCloud Runを検討したほうがよい。
* [Cloud Run](./gcp_cloud_run.md)を参照。


## BigQuery
* カラム型(列指向)のデータウェアハウス。集計が速い。
* 処理を分散させることで高速化している。
* 例えば、社内の各部門が参照する集計データをここで作る、といったユースケースに向いていそう(IMO)。
* TODO: 未整理。
