---
title: "システム・OS - Linux"
updated: 2026-08-29
---

[TOP(About this memo))](../README.md) > [一覧(Linux)](./README.md) > システム・OS

## OSの確認
* Red Hat系
    * `cat /etc/redhat-release`
* Amazon Linux
    * `cat /etc/system-release`
    * `cat /etc/os-release`
* Ubuntu、Debian
    * `cat /etc/os-release`
    * 出力例
        ```
        VERSION="24.04.1 LTS (Noble Numbat)"
        ID=ubuntu
        ID_LIKE=debian
        PRETTY_NAME="Ubuntu 24.04.1 LTS"
        ```
    * `ID_LIKE=debian` とあるとおり、UbuntuはDebian系のディストリビューション。
* `/etc/os-release` はsystemdが定めた形式で、現在はほとんどのディストリビューションが持っている。ディストリビューションを問わず確認したい場合はまずこれを見るのが早い。

## CPUアーキテクチャの確認
* `arch` コマンド、もしくは `uname -m`
    * `x86_64` や `amd64` と出たら64ビットのx86系
    * `i386` や `i686` だったら32ビットのx86
    * ARMなら `aarch64` や `arm64`、`armv7l` などarmを含んだ文言が出る
* macOSでは、Rosetta 2経由で特定のアーキテクチャとして実行した結果を確認することもできる。
    ```
    uname -m
    arch -arm64e uname -m
    arch -x86_64 uname -m
    ```
* 用語
    * amd64 … x86を64ビットに拡張したもの。この命令セットをAMDが先に世に出した経緯から、Intel製CPU向けであってもamd64版と表記される。一般的なPC・サーバーはこれ。x64、x86_64も同じものを指す。
    * arm64（aarch64） … ARMの64ビットアーキテクチャ。Apple Silicon、AWS Graviton、Raspberry Pi 3以降の64ビットOSなど。
    * armhf … ARM上で動作する32ビット版のうち、ハードウェア浮動小数点演算命令を使うもの。

## サービスの管理
* 現在の主要ディストリビューションはinitシステムとしてsystemdを採用しており、`systemctl` でサービスを操作する。CentOS専用のコマンドではない。
* 古いDebian系やUpstart時代の名残として `service` コマンドも残っており、内部的にsystemdへ委譲される。
    ```
    sudo systemctl restart apache2
    sudo service apache2 restart   # 上と同等（互換のため残っている）
    ```
* 状態の確認
    * `systemctl list-unit-files | grep chronyd`
        * `enabled` になっているものは自動起動の対象。
    * `systemctl list-units --type=service`
        * 現在ロードされているサービスの一覧。
    * `systemctl status <サービス名>`
* Upstartを使っていた頃のUbuntuでは `service --status-all` で一覧を確認できた。
    * `[+]` 稼働している / `[-]` 停止している / `[?]` 稼働状況を判断できない

## dmesg
* カーネルリングバッファのデータを表示するコマンド。カーネルやユーザープロセスのメッセージを確認できる。
* オプションを付けずに実行すると、カーネル起動時からのメッセージが表示される。メッセージの左端はカーネル起動時からの相対時刻。
* USBメモリなどデバイスの認識確認に使える。
```
dmesg | grep some-device
```

## システムコール
* システムコールとは、OSが提供する機能をアプリケーションが利用するための仕組み。
* アプリケーションの動作のうち重要なものは、ほぼ全てシステムコールを介して実現されている。
    * ネットワークを利用した通信
    * ファイルへの入出力
    * 新しいプロセスの生成
    * プロセス間通信
    * コンテナの生成
    * write、mmap、fork など
* メリット
    * ハードウェアを操作するためのシンプルなインターフェースが提供される
    * アプリケーションが安全かつセキュアにOSのリソースを利用できる
* fork
    * プロセスから別のプロセスを生成するシステムコール。
    * 例）あるプロセスIDを持つhttpdが、別のプロセスIDとなるhttpdプロセスを生成する。

## ローカライゼーション
* `locale` コマンドでローカライゼーション系の環境変数を確認できる。
* `LANG` 変数を変えることで、関連する変数をまとめて切り替えられる。
    * 例）`LANG=C` で英語（Cロケール）になる。
* 現在の設定は `echo $LANG` で確認できる。
* (参考) https://eng-entrance.com/linux-localization-lang

## ログの保存場所
* ログファイルは `/var/log` 以下に配置するようFHS（Filesystem Hierarchy Standard）で規定されている。FHSに準拠したシステムであれば、ログファイルは `/var/log` 以下に置かれる。
* 自前のログを置く場合も `/var/log` 以下にしておくのが無難。
* ディレクトリ構造の全体像は[ディレクトリ構造](./linux_directory.md)を参照。
