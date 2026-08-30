---
title: "ネットワーク関連コマンド - Linux"
updated: 2026-08-29
---

[TOP(About this memo))](../README.md) > [一覧(Linux)](./README.md) > ネットワーク関連コマンド

ネットワークそのものの仕組み（プロトコル、DNS、証明書など）については[ネットワーク](../network/README.md)を参照。ここではLinux/macOS上で使うコマンドを扱う。

## 手早く確認したいとき
* グローバルIPアドレス
    * `curl -s ifconfig.me`
* 静的に定義されたホスト名の対応
    * `cat /etc/hosts`

## ifconfig、ip
* LinuxやmacOSなど、主にUNIX系OSで用いるネットワーク環境の状態確認・設定のためのコマンド。
* ホストに設置された有線LANや無線LANなどのネットワークインターフェースに対し、IPアドレスやサブネットマスク、ブロードキャストアドレスなどの基本的な設定ができる。加えて、現在の設定を確認できる。
* Linuxでは `ifconfig`（net-toolsパッケージ）が非推奨となっており、`ip` コマンドへの移行が推奨されている。RHEL 7以降をはじめ、多くのディストリビューションでnet-toolsは既定でインストールされない。
    * `ip addr` … インターフェースとIPアドレスの一覧（`ifconfig` 相当）
    * `ip link` … インターフェースの状態
    * `ip route` … ルーティングテーブル（`route -n` 相当）
* macOSでは現在も `ifconfig` が標準のコマンド。
* Windows環境では `ifconfig` ではなく `ipconfig`。

### 表示される情報の概要
* 一番先頭に出ているのがネットワークインターフェース名。
* `flags=...` はインターフェースの状態を表すフラグ。
* `ether` に続くのがMACアドレス（インターフェースの固有アドレス）。
* `inet` が現在のIPv4アドレス（静的に設定したものかDHCPによる自動割り当て）。その後の `netmask` はサブネットマスクで、IPアドレスのどこまでがネットワーク部でどこからがホスト部かを定義する。
* `inet6` が現在のIPv6アドレス。
* `RX` は受信について、`TX` は送信についての情報（パケット数、エラー数など）。
* (参考) https://qiita.com/pe-ta/items/aff8db72530c6baa11b2

### インターフェース名の例
* `lo0`（loopback）
    * ネットワークのテストに使えるよう用意された仮想インターフェース。NICがなくてもこれは表示される。`127.0.0.1` がIPアドレスとして自動で割り当てられる。
* `gif0`（generic tunnel interface）
    * IPv6/IPv4トンネリングを行うときに使うインターフェース。
* `stf0`（6to4 tunnel interface）
    * gifがIPv6パケットをIPv4パケットにカプセル化する（およびその逆を行う）のに対し、これはIPv6パケットをIPv4ネットワークにルーティングするためのインターフェース。
* `en0`（Ethernet 0）
    * `en` の後に続く番号は認識した順に割り振られる。手元のマシンのIPアドレスとMACアドレスは通常このインターフェースに表示される。
    ```
    ether xx:xx:xx:xx:xx:xx
    inet 192.168.0.2 netmask 0xffffff00 broadcast 192.168.0.255
    ```
    * MACアドレスの先頭3オクテットはベンダーID（OUI）で、IEEE Registration Authorityの公開リストから端末のベンダーを調べることができる。検索する際はコロンをダッシュに変換する。
* `bridge0`
    * 仮想インターフェースを外部ネットワークに接続するためのブリッジ。
* `vboxnet0`、`docker0`
    * それぞれVirtualBox、Dockerが作成する仮想インターフェース。
* macOSでは、ネットワークユーティリティやシステム設定からも各インターフェースの情報を確認できる。

## netstat、ss
* CentOS 7およびRHEL 7以降、ネットワークに関連した一部のコマンドは非推奨となり、デフォルトではインストールされなくなった。`netstat` もその一つで、代替は `ss` コマンドになる。
    * 出力形式が異なるため、netstat前提のスクリプトはそのままでは動かない。どうしてもnetstatが必要な場合は明示的に導入する。
    ```
    yum install net-tools
    ```
    * (?) かつては ss 側の不具合が指摘されることもあったが、現在は ss が標準と考えてよい。新規に書くなら ss を使う。
* netstatの例
    * `netstat -nl`
        * 接続待ち状態のソケットを確認できる（コンテナに入って、どのポートがLISTENされているか確認する場合など）。
        * `-l` 接続待ち（LISTEN）状態のソケットのみを表示
        * `-n` 名前解決をせずに数字で表示
    * `netstat -tanp | grep -i listen`
    * `netstat -rn` … ルーティングテーブルの確認
* ssの例
    * `ss -t` … TCPソケットを表示
    * `ss -t4` … IPv4のソケット（コネクション確立済（ESTABLISHED）のみ表示）
    * `ss -lt4` … リスニングソケット（LISTEN）を表示
    * `ss -at4` … リスニング状態も含めて全て表示
    * `ss -nt4` … 名前解決をせずポート番号を数字で表示
    * `ss -pt4` … ソケットを使っているプロセスを表示
    * `ss -u` … UDPソケットを表示
    * `ss -lnu4` … UDPパケットの到着を待っているソケットを確認
    * (参考) https://qiita.com/hana_shin/items/632b3a1eb44bf84e94f7

## lsof
* プロセスが開いているファイル（ソケットを含む）を一覧するコマンド。
* 特定のポートを使っているプロセスを探す。
```
sudo lsof -i:8080 -P
```
```
java    4190 username  147u  IPv6 0x372f84f782df4f69      0t0  TCP localhost:8080 (LISTEN)
```
* `-P` を付けないと、ポート番号がサービス名に変換されて表示される。
* その他の例
    * `lsof -i -P` … ネットワーク関連の全てを表示
    * `lsof -i @localhost -P`
    * `lsof -iTCP:22 -P`
    * `lsof -c nginx` … nginxというプロセスが開いているファイルを確認
    * `lsof -i -P | grep LISTEN` … LISTEN中のものだけ表示（`netstat -nat | grep LISTEN` でも同様）
    * `lsof -i -P | grep ESTABLISHED`
    * `lsof /path/to/mountpoint` … 指定したパス配下を使っているプロセスを確認（アンマウントできない原因の特定などに使う）
* アドレスが `*` の場合は `0.0.0.0` なので、同じネットワーク上のマシンからアクセスできる状態になっている。
    * 例えばDockerコンテナのポートをマッピングしている場合、デフォルトでは `*:ポート` になる。
* (参考) https://paulownia.hatenablog.com/entry/2018/07/31/235913
* macOSでは `rapportd` というプロセスが表示されることがあるが、これはiPhone等とのハンドオフ通信用のサーバープロセス。

## (WIP) nmap
* ポートスキャンによく使われるツール。
* パッケージマネージャ（yum/dnf、Homebrew等）でインストールする必要がある。
* 自分が管理していないホストへのポートスキャンは、法的・契約的な問題になり得るため行わないこと。

## curl / wget
* `curl -sS http://example.com`
    * `-s` 進捗を表示しない / `-S` エラーは表示する
* wgetとの違い
    * curlはFTP、FTPS、HTTP、HTTPS、SCP、SFTP、TFTP、TELNET、LDAP、FILE、POP3、IMAP、SMTP、RTMPなど多数のプロトコルに対応する。
    * wgetはHTTP・HTTPS・FTPのみだが、ファイルを再帰的（`-r`）にダウンロードできる。

### POSTやヘッダーを付ける
* JSONファイルを添付して送る
```
curl -X POST -H "Content-Type: application/json" \
     --data-binary @request.json \
     https://api.example.com/echo | jq
```
* インラインでJSONを送る
```
curl -H "Content-Type: application/json" -d '{"email":"user@example.com","password":"password"}' http://localhost:8081/v0.1/login
```
* `Content-Type` を指定しないと `application/x-www-form-urlencoded` として扱われる。
```
curl -d email=user@example.com -d password=password http://localhost:8081/v0.2/login
```
* クエリパラメータ・カスタムヘッダーを付ける
```
curl -X GET "https://api.example.com/v1/items?id=2&lng=139.0000&lat=35.0000" \
     -H "accept: application/json" \
     -H "X-API-Key: <APIキー>"
```
* multipart/form-dataでファイルを送る
```
curl -X POST "https://api.example.com/v1/items" \
     -H "accept: application/json" \
     -H "X-API-Key: <APIキー>" \
     -H "Content-Type: multipart/form-data" \
     -F "id=2" -F "content=sample" \
     -F "image=@sample.png;type=image/png"
```
* APIキーやトークンをコマンドラインに直接書くと、シェルの履歴（`~/.bash_history` 等）や `ps` の出力に残る。環境変数や `--config` で渡すか、少なくとも履歴に残さない運用にする。

### ヘッダー情報の出力
* `-i` … レスポンスヘッダーとボディの両方を出力する。通常はこちらを使う。
* `-I` … HEADリクエストを送る。リクエストメソッドの指定を含んでいるため、`-d` と併用すると `You can only select one HTTP request method!` というエラーになる。
* `-v` … リクエスト・レスポンス両方のヘッダーを含む詳細を出力する。

### メソッドの指定
* `-X <メソッド>` で指定する。
    * ただし `-d` を付けた時点で自動的にPOSTになるなど、暗黙のデフォルトがある。意図しないメソッドにならないよう、明示的に指定した方が分かりやすい。
