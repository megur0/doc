---
title: "パッケージ管理 - Linux"
updated: 2026-08-29
---

[TOP(About this memo))](../README.md) > [一覧(Linux)](./README.md) > パッケージ管理

系統ごとの違いについては[ディストリビューション](./linux_distribution.md)を参照。

## パッケージマネージャ
* Red Hat系は `yum` / `dnf`、Debian系は `apt` / `apt-get` がメジャー。
    * RHEL 8以降やAmazon Linux 2023では `dnf` が正式なコマンドで、`yum` はそのエイリアスとして残っている。
* より低レベルなコマンドとして `rpm` や `dpkg` もある。
    * これらは依存関係の解決を自分で行う必要があるため、通常は使わない。
    * `yum`/`dnf` は `rpm` を、APT系コマンドは `dpkg` を、それぞれ依存解決やリポジトリ管理の機能で包んだものにあたる。

## EPELリポジトリ（Extra Packages for Enterprise Linux）
* RHEL系ディストリビューションにおける、拡張パッケージのサードパーティ・リポジトリ。
* サードパーティ・リポジトリを使う理由
    * 「使いたいアプリケーションが標準のリポジトリに含まれていない」もしくは「含まれていてもバージョンが古い」ため。
    * これはディストリビューションのサポート方針（長期にわたって安定したバージョンを維持する）に起因する。
* RHEL系をクラウドやオンプレミスで使う際は、`epel-release` をインストールするのが基本になる。
* (参考) https://qiita.com/yamada-hakase/items/fdf9c276b9cae51b3633

## rpm
* Red Hat系のLinuxディストリビューションで使われている RPM（Red Hat Package Manager）パッケージを扱うパッケージ管理コマンド。
* RPMデータベースを参照することで、現在システムにインストールされているパッケージやそのバージョンが分かる。
* 主なオプション
    * `-i` インストール / `-U` アップグレード / `-v` 情報表示を増やす / `-h` インストール時の経過を `#` マークで表示
    * `rpm -qilp <パッケージファイル名>` … パッケージの情報とインストールされるファイルを表示
    * `rpm -qRp <パッケージファイル名>` … 指定したパッケージの依存先を調べる
    * `rpm -qa` … インストール済みの全RPMパッケージを表示
    ```
    rpm -qa | egrep ^openssl-
    ```

## yum / dnf
* RPMデータベースを参照するパッケージ管理システム。
* `yum list installed | grep <パッケージ名>` … インストールされているパッケージを確認
* `yum list available | grep <パッケージ名>` … インストール可能なパッケージを確認
* `yum list updates | grep <パッケージ名>` … アップデート可能なパッケージを確認
* `sudo yum install <パッケージ名>`
* (参考) リポジトリ一覧 https://qiita.com/bezeklik/items/9766003c19f9664602fe

## apt / apt-get
* Debian系（Ubuntu等）で使う。
* `-y` で全ての確認にyesと答える。
* 現在は、対話的な利用には `apt` が推奨されている。`apt-get` は出力形式が安定しているため、スクリプト内での利用に向いている。

### パッケージのアップデート
* `sudo apt update`
    * インストール可能なパッケージの「一覧」を更新する。実際のパッケージのインストール、アップグレードは行わない。
* `sudo apt upgrade`
    * インストール済みのパッケージを新しいバージョンにアップグレードする。
    * 更新された一覧をもとに実行されるため、`apt update` と組み合わせて使う必要がある。
    * (IMO) 全てのパッケージをまとめて更新するため、本番サーバーでは影響範囲が読みにくい。個別に更新する方が安全なケースが多い。
* 個別にアップデート
    ```
    sudo apt list --upgradable
    sudo apt install --only-upgrade <パッケージ名>
    ```

### PPA（Personal Package Archive）
* Ubuntu公式にはサポート（審査）されていないパッケージを配布する仕組み。個別にリポジトリを追加することでインストールできる。
* 提供元を信頼できるかを確認した上で使う。
```
sudo add-apt-repository ppa:<ユーザー名>/<リポジトリ名>
sudo apt update
sudo apt install <パッケージ名>
```

## macOSのHomebrew
* [Homebrew](../mac/mac_homebrew.md)を参照。

## よく使われるパッケージ
* tmux
    * 一つのターミナルを分割して複数のペインで作業できる。SSH接続先でも分割して使える。
    * セッションを維持できるため、接続が切れても作業を継続できる（[screen](./linux_process.md)と同様の用途）。
* gnupg
    * 通信とデータ保存を安全なものにするGNUツール。データの暗号化とデジタル署名の作成に使うことができる。
    * 先進的な鍵管理機能を持ち、RFC4880で定められたOpenPGP標準に準拠している。
    * `gnupg2` は移行用のダミーパッケージであり、`gpg2` から `gpg` へのシンボリックリンクを提供する。
    * 関連するパッケージ: dirmngr
    * (参考) https://qiita.com/karkwind/items/9cb2907060f210fc940a
* dbus
    * プロセス間通信の仕組み。
        * システムバスは、システムに1つ存在し、すべてのユーザーがアクセスできる。システムの稼働に利用される。
        * セッションバスは、ユーザーごとに存在し、そのユーザーのみが利用する。
        * サービス、オブジェクト、インターフェースという単位で構成される。
    * (参考) https://qiita.com/byuu/items/c600366b9c138f639863
* htop
    * `top` コマンドより高機能なプロセスビューア。
* build-essential
    * 名前の通り、開発に必須のビルドツールを提供するパッケージ。gcc（GNU C Compiler）、g++（GNU C++ Compiler）、makeなどが入る。
