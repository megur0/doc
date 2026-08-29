---
title: "Cloud Storage - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > Cloud Storage


## 料金体系
* https://cloud.google.com/storage/pricing


## ストレージクラス
* https://cloud.google.com/storage/docs/storage-classes
* ストレージクラスは、すべてのオブジェクトで使用されるメタデータ。
* バケットに対してデフォルトのストレージクラスを設定できる。
* オブジェクトの可用性と料金に影響する。
* Standard, Nearline, Coldline, Archive
    * 右に行くほど保存の費用は安くなり、取り出しの費用が高くなる。
    * 右に行くほど最小保存期間が長くなる。
* 最小保存期間
    * 最小保存期間を迎える前に削除・置換・移動をしたとしても、残りの期間分の費用は請求される。
    * つまり、以下の料金は同じになる。
        * 最小保存期間の到来前に削除
        * 最小保存期間の満了時に削除
    * https://cloud.google.com/storage/pricing-examples#early-delete

### Autoclassの設定
* https://cloud.google.com/storage/docs/using-autoclass
* バケットにAutoclass機能を設定できる。
* 設定すると、バケット内のオブジェクトがアクセスパターンに基づいて適切なストレージクラスへ自動的に移行される。


## バケットの作成
* https://cloud.google.com/storage/docs/creating-buckets#command-line


## アクセス管理
* https://cloud.google.com/storage/docs/access-control
* 2つの管理方法が提供されている。
    * 均一(推奨)
        * IAM(Identity and Access Management)のみを使用して権限を管理する。
        * コマンドラインではバケットの生成・更新時に`--uniform-bucket-level-access`を指定することで有効化できる。
    * きめ細かい管理
        * IAMとアクセス制御リスト(ACL)を併用して権限を管理する。


## バージョニング
* https://cloud.google.com/storage/docs/object-versioning
* 削除されたオブジェクトを、バージョニングされた非現行バージョンのオブジェクトとして保持する機能。
* デフォルトでオフになっている。


## ライフサイクルの設定
* https://cloud.google.com/storage/docs/managing-lifecycles#set
* 一定期間経過後に削除する、といったアクションを実行させる管理機能。
* isLive
    * https://cloud.google.com/storage/docs/lifecycle#islive
    * オブジェクトのバージョニングを有効にしている場合は、対象がライブバージョンである場合に条件を満たす。
    * 無効にしている場合は、`isLive`がtrueのときすべてのオブジェクトがライブバージョンとみなされる。
        * つまりバージョニングを無効にしているときは、`isLive`はtrueにしておけばよい、と理解している(?)。
