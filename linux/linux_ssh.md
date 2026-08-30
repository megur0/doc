---
title: "SSH - Linux"
updated: 2026-08-29
---

[TOP(About this memo))](../README.md) > [一覧(Linux)](./README.md) > SSH

## 基本
* SSHでは、以下の点で従来のTelnetより安全な通信が行える。
    * パスワードやデータを暗号化して通信する。
    * クライアントがサーバーに接続する時に、接続先が意図しないサーバーに誘導されていないか厳密にチェックする（ホスト鍵の検証）。
* 認証方式は大きく2種類。
    * パスワード認証方式
        * デフォルトの認証方式で、ユーザー名とパスワードでログインする。
        * ユーザー名とパスワードは接続先OSのユーザーアカウントの情報が使用される。
    * 公開鍵認証方式
        * 公開鍵と秘密鍵の2つの鍵（キーペア）を使用した接続方式。
        * サーバーに公開鍵、クライアントに秘密鍵を置いて使用する。
        * パスワード入力なしでログインすることが可能になる。
    * パスワード認証方式はサーバー側で明示的に無効にしていない限り使用できるが、総当たり攻撃に晒されるためセキュリティ上は脆弱で、無効にしていることが多い。共有レンタルサーバーなどではパスワード認証のみ提供されているケースもある。

### SFTPのパスワード認証と鍵認証の違い
* どちらも通信内容は暗号化される。
* ただし鍵認証の場合はパスワードがネットワークを流れないため、より安全。
* どちらの場合も、加えて接続元IPを制限するなどの対策を取る方が望ましい。

## 主なコマンド（OpenSSH）
* `ssh`
    * `ssh <ユーザー名>@<ホスト> -i ~/.ssh/id_ed25519`
    * `-i identity_file` … 公開鍵認証で使用する秘密鍵ファイルを指定する。指定しない場合は `~/.ssh/id_ed25519`、`~/.ssh/id_rsa` などが順に試される。
    * `-F configfile` … 設定ファイルを指定する。デフォルトでは `~/.ssh/config` が使用される。
    * `-p` … ポート番号を指定する。
* `scp` … SSHを使用してリモートホストとの間でファイルを転送する。
* `ssh-keygen` … 鍵ペアの生成。
* `ssh-copy-id` … 公開鍵をリモートホストに登録するコマンド。環境によってはインストールされていない場合がある。

## 鍵の配置パターン
* 一般的なサーバーへのログイン
    * SSHサーバー（公開鍵: `~/.ssh/authorized_keys`）　←→　SSHクライアント（秘密鍵: `~/.ssh/<秘密鍵>`）
* サーバーからGitホスティングサービス等へアクセスする場合
    * SSHサーバー側が「クライアント」になる。サーバーに秘密鍵を置き、公開鍵をGitホスティングサービス側に登録する。
    * `~/.ssh/config` で接続先ホストに対して使用する秘密鍵を指定しておく。
* (参考) https://qiita.com/ir-yk/items/af8550fea92b5c5f7fca

## パスワード認証方式を無効にする
* クラウドが提供する既定のイメージでは、最初から無効になっていることが多い。
* 設定方法
    ```
    sudo vi /etc/ssh/sshd_config
        PasswordAuthentication no
        KbdInteractiveAuthentication no
    ```
    * `KbdInteractiveAuthentication` は、OpenSSH 8.7（2021年）より前は `ChallengeResponseAuthentication` という名前だった。古い記事では後者で書かれていることが多い。旧名も非推奨のエイリアスとして残っているため設定ファイルはそのまま動くが、新しく書くなら `KbdInteractiveAuthentication` を使う。
    * 設定を反映するために再起動する前に、**必ずターミナルの別ウィンドウでSSH接続できることを確認する**。既存のセッションを閉じてしまうと、接続手段を失う。
    * 再起動
        ```
        sudo systemctl restart sshd
        ```
        * systemd以前のOS（CentOS 6等）では `sudo service sshd restart`。

## scp
* scpは、sshを使ってネットワーク・ホスト間でファイルを安全にコピーするためのコマンド。
* リモート → ローカル
    ```
    scp -i ~/.ssh/secret.pem -r <ユーザー名>@<ホスト>:/remote/path /local/path
    ```
* ローカル → リモート
    ```
    scp -i ~/.ssh/secret.pem -r /local/test.txt <ユーザー名>@<ホスト>:/home/user/tmp/
    ```
* `-r` を付けるとディレクトリごと再帰的にコピーする。
* `~/.ssh/config` にホストを登録しているなら `scp <ホスト名>:/path /local/path` でよい。
* (参考) https://qiita.com/katsukii/items/225cd3de6d3d06a9abcb
* なお、OpenSSHのscpは内部実装がSFTPベースに置き換えられており、公式には `sftp` や `rsync` の利用が推奨されている(?)。用途によっては[rsync](./linux_file.md)の方が便利（`--exclude` などが使える）。

## 鍵の作成
* 現在はRSAよりed25519が推奨される。鍵長が短く、生成・検証が速く、安全性も十分とされている。
    * OpenSSH 8.8（2021年）以降、SHA-1を用いる `ssh-rsa` 署名がデフォルトで無効化された。RSA鍵自体が使えなくなったわけではなく、SHA-2を使う `rsa-sha2-256` / `rsa-sha2-512` であれば利用できるが、新規に作るならed25519が無難。
* ed25519の例
    ```
    ssh-keygen -t ed25519 -C admin@example.com -f ~/.ssh/example/id_ed25519 -N "" && chmod 600 ~/.ssh/example/id_ed25519 && cat ~/.ssh/example/id_ed25519.pub
    ```
* RSAを使う場合の例
    ```
    ssh-keygen -t rsa -b 4096 -C admin@example.com -f ~/.ssh/example/id_rsa -N ""
    ```
* オプション
    * `-t` … 作成する鍵の形式を `rsa`、`ecdsa`、`ed25519` などから指定する。
    * `-C` … コメントを指定する（デフォルトは「ユーザー名@ホスト名」。`-C ""` でコメントを削除）。
    * `-b` … 鍵長。RSAの場合は3072ビット以上が推奨される。ed25519は鍵長が固定のため指定不要。
    * `-f` … 出力先のパス。
    * `-N` … パスフレーズ。空文字を指定するとパスフレーズなしの鍵となる。パスフレーズは、秘密鍵ファイルを盗まれた際にすぐには使えないようにするためのもの。
* 秘密鍵は `chmod 600` にしておかないと、ssh実行時にエラーになる。
* 秘密鍵から公開鍵を再生成する
    ```
    ssh-keygen -y -f /path/to/my-key-pair.pem > my-key-pair.pub
    ```

## ~/.ssh/config の設定
* SSH接続が途中で切れないようにする設定。サーバー側が無通信の接続を自動的に切ってしまうため、クライアント側では定期的にパケットを送るようにする。
```
    ServerAliveInterval 300
    TCPKeepAlive yes
```
* 踏み台サーバー（`ProxyJump`）の設定もここで行える。
    * (参考) https://qiita.com/oohira/items/7deae31469cfbbd740c1
* 設定オプション全般
    * (参考) https://qiita.com/ryysud/items/f799b946c9f65de363cc

## サーバーからGitホスティングサービスへSSH接続する場合の設定例
```
mkdir ~/.ssh/example
ssh-keygen -t ed25519 -C admin@example.com -f ~/.ssh/example/id_ed25519 -N "" && cat ~/.ssh/example/id_ed25519.pub
# 表示された公開鍵をGitホスティングサービス側（GitHubならDeploy keys）に登録しておく
chmod 400 ~/.ssh/example/id_ed25519
vi ~/.ssh/config
    Host example
    HostName github.com
    User git
    IdentityFile ~/.ssh/example/id_ed25519
chmod 600 ~/.ssh/config
```

## ssh-agent
* 秘密鍵のパスフレーズを都度入力しなくて済むよう、鍵をメモリ上に保持しておく仕組み。
* (IMO) パスフレーズを空にしておけば同じことができるため、個人の開発用途では導入する動機が薄い。パスフレーズは「秘密鍵ファイルが盗まれた場合」の保険であり、それを前提にした上でパスフレーズをエージェントに預けるのは、目的と手段がややちぐはぐに感じる。以下のような扱いづらさもある。
    * ssh-agentを起動したシェル内でのみ有効。ログアウトしたりシェルを変更すると使えなくなり、再実行が必要になる。
    * `ssh-add` による秘密鍵とパスフレーズの登録が、ssh-agentを起動するたびに必要。
    * ssh-agentは自動終了しない。ログアウトやシェルの変更後に新しいシェルでssh-agentを起動すると、多重起動が発生する。
    * エージェント転送（`-A`）を使うと接続先のサーバーから手元の鍵が利用可能な状態になるため、接続先が信頼できない場合はセキュリティリスクになる。
* (参考) https://www.server-memo.net/server-setting/ssh/ssh-agent.html
* なお、macOSではKeychainと連携して起動時に鍵を読み込ませることができ（`UseKeychain yes`）、この場合は上記の煩雑さの多くが解消される。
