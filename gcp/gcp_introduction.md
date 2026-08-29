---
title: "導入・基本 - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > 導入・基本


## 公式ドキュメント
* ドキュメントのトップ
    * https://cloud.google.com/docs
* Cloud SDK(gcloud)リファレンス
    * https://cloud.google.com/sdk/gcloud/reference
    * 基本的には`--help`で出てくる内容と同等。内部でどのAPIをどう呼び出しているか、まではドキュメント化されていない(?)。
    * 内部の挙動を追いたい場合は、後述のREST APIリファレンスを辿るほうが早い。
* REST APIリファレンス
    * 例: Cloud Build であれば https://cloud.google.com/build/docs/api/reference/rest


## ローンチステージ
* https://cloud.google.com/products?hl=ja#product-launch-stages
* Preview
    * SLAやテクニカルサポートの確約の対象にならない点に注意。
    * プレビューステージの平均期間は約6か月間とされている。
    * ただし、実際には長期間GAにならないサービスも多い(IME)。たとえばCloud Runのドメインマッピングは長くPreviewのままとなっている。
* General Availability(GA)
    * 該当する場合はGoogle Cloud SLAの対象となる。
* Deprecated
    * スケジュールに沿って提供終了・削除される。


## VPC
* https://cloud.google.com/vpc/docs/using-vpc
* (参考) https://cloud-ace.jp/tech_blog/gcp-init/

### プロジェクト作成直後にやっておくとよいこと(IMO)
* プロジェクトを作成すると`default`という名前のVPCネットワークが自動作成されている。これは削除しておくのが無難。
    * このネットワークは全てのリージョンにサブネットワークが作成されており、ファイアウォールの設定もかなり緩い。
* VPCネットワークが無いとVMインスタンスを起動できないため、汎用的なものを1つ作成しておく。
    * 例) プロジェクト名と同じ名前のVPCネットワークを作成し、東京リージョンに/26〜/25程度のサブネットワークを作成する。
* インターネットに出られないと困るケースも多いため、上記で作成したVPCネットワークにCloud NATを設定しておく。
