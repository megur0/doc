---
title: "セキュリティ(IAP・Cloud Armor・IDトークン) - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > セキュリティ

* WIP。Cloud Armorまわりは特に未整理。

## 参考
* (参考) https://qiita.com/szk3/items/9814b5e6e20d03e67baf


## IAP (Identity-Aware Proxy)
* https://cloud.google.com/iap/docs/concepts-overview
* Google Cloudのコンピューティングサービス(GCE、GAE、GKE、Cloud Runなど)へのアクセス制御を行う、ID認証プロキシサービス。
* GoogleのID認証を経てアプリケーションへアクセスできるようになり、セキュアな運用が可能となる。
* VPNを利用するより遥かに安い。
* 原則無料だが、GCEやGKE(ノードでGCEを利用)と併用すると有料になるとのこと(?)。

### GAEでのアクセス制御
* (参考) https://yamavlog.com/traial-gcp-cloud-iap/

### 外部IPアドレスがないGCEインスタンスにCloud IAP TCP Forwardingを設定する
踏み台サーバーなしでSSH接続できるようにする構成。

* 対象のプロジェクトにおいて、対象ユーザーのIAMにIAPへアクセスできるロールをバインドする。
* ファイアウォール設定で、Cloud IAPのバックエンドIPアドレス範囲`35.235.240.0/20`をVMのTCP Port 22に対して許可する。
    * このIPレンジはGoogleが公開している固定のものなので、そのまま指定してよい。


## Cloud Armor
* WAF(Web Application Firewall)。
* https://cloud.google.com/security/products/armor
* TODO: 未整理。


## OAuthで受け取ったIDトークンの扱い
* 有効期限をチェックせずにそのまま利用するのはNG。
    * トークンが実質的に永続して使えてしまうため、盗まれた場合にずっと利用され続けてしまう。
    * ログアウトした後もトークンが利用可能な状態になってしまう。
* IDトークンの検証はライブラリ(Firebase Admin SDK等)側で行われるのが一般的なので、自前で有効期限のチェックを飛ばすような実装をしないこと。
* (参考) Google SignInするSPAとGoサーバー間のセッション管理
    * https://bati11blog.hatenablog.com/entry/2018/06/25/001711
