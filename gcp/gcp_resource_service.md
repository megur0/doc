---
title: "リソース・プロジェクト・Cloud APIs - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > リソース・プロジェクト・Cloud APIs


## リソース
* https://cloud.google.com/docs/overview
* https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy#projects
* Google Cloudは、コンピュータやハードディスクドライブなどの物理アセット一式と、仮想マシン(VM)などの仮想リソースで構成される。
* 各データセンターはリージョンにある。
* 各リージョンは、リージョン内で互いに隔てられたゾーンの集合となる。
* 例) 東アジアのリージョンのゾーンaは`asia-east1-a`。

### サービス
* サービス = ソフトウェアやハードウェア製品と見なして利用できるもの。
* サービスは、基本となるリソースへのアクセスを提供する。

### プロジェクト
* 割り当てて使用するGoogle Cloudのリソースはすべて、プロジェクトに属している必要がある。
* プロジェクトとは、構築する対象をまとめて管理する1つの単位。
* 設定や権限に加え、アプリケーションに関する情報を記述したその他のメタデータで構成される。

### リソースの階層
* 組織 > フォルダ > プロジェクト > リソース
* ポリシーは階層に沿って継承される。


## Cloud APIs
* Google Cloud APIs = Google Cloudの各サービスへのプログラマティックインターフェース。
* コンピューティングからネットワーキング、ストレージ、機械学習ベースのデータ分析まで、あらゆる機能をアプリケーションへ追加できる。
* `googleapis.com`の1つ以上のサブドメインで動作する。
* JSON HTTPとgRPCの両方のインターフェースをクライアントに提供する。
* すべてのCloud APIは、直接呼び出すか、Google APIクライアントライブラリ経由で呼び出すことができる。
* コンソールからの実行にしても、CLIにしても、Terraformにしても、内部的にはこのAPIを使っている。

### CLIの挙動をAPIリファレンスから辿る
* ドキュメントには「基盤となるREST APIリクエストがこのタスクでどのように作成されるかについては、〜ページのAPI Explorerをご覧ください」という記述が各所にある。
    * 例: https://cloud.google.com/sql/docs/postgres/create-instance#terraform
* そのため、CLIに渡す引数が最終的にどう動作するかは、APIリファレンスを辿れば把握できるはず。

### gRPCとRESTの関係
* APIの実体はgRPCだが、APIリファレンスを見ると形式がRESTになっており、「The URL uses gRPC Transcoding syntax.」と書かれている。
* これはgRPCトランスコーディングによってHTTP / JSONとして利用できるようにしている、ということ(?)。
    * 関連: https://cloud.google.com/endpoints/docs/grpc/transcoding


## プロジェクトでデフォルトで有効になるAPI
* プロジェクト作成時点でデフォルトで有効になっているAPI(2023年頃時点。構成は変わりうる)。
    * BigQuery API
    * BigQuery Migration API
    * BigQuery Storage API
    * Cloud Datastore API
    * Cloud Logging API
    * Cloud Monitoring API
    * Cloud SQL
    * Cloud Storage
    * Cloud Storage API
    * Cloud Trace API
    * Google Cloud APIs
    * Google Cloud Storage JSON API
    * Service Management API
    * Service Usage API
* `gcloud projects create`に`--no-enable-cloud-apis`をつけると、これらが有効にならない。
