---
title: "日時・NTP - Linux"
updated: 2026-08-29
---

[TOP(About this memo))](../README.md) > [一覧(Linux)](./README.md) > 日時・NTP

## 基本的なコマンド
* `date` … 日付・時刻の表示
    * `date "+%Y%m%d_%H%M%S"` のようにフォーマットを指定できる。
* `time <コマンド>` … コマンドの実行時間を計測
* `timedatectl` … systemd環境でのタイムゾーン・時刻同期状態の確認と設定

## NTPによる時刻同期（chrony）
* 現在の主要ディストリビューション（RHEL 7以降、Amazon Linux 2 / 2023、Ubuntuのサーバー用途など）では、NTPクライアントとしてchronyが標準的に使われる。古い `ntpd` から置き換えられた。
* AWS環境では、Amazon Time Sync Service（リンクローカルアドレス経由で提供されるNTPサーバー）を参照する設定が既定で入っていることが多く、追加設定なしでも同期される。

### インストール状況の確認
```
yum list installed | grep chrony
systemctl list-unit-files | grep chronyd
timedatectl
```
* `timedatectl` の出力で `NTP service: active`（環境によっては `NTP enabled: yes`）を確認する。

### インストール
```
yum list available | grep chrony
sudo yum install chrony
```
* Amazon Linux 2023やRHEL 8以降では `yum` ではなく `dnf` を使う（`yum` はdnfへのエイリアスとして残っている）。

### 設定
* 設定ファイルは `/etc/chrony.conf`。編集する前にバックアップを取っておく。
```
sudo cp -a -p /etc/chrony.conf /etc/chrony.conf.bak
sudo vi /etc/chrony.conf
```
* (IME) クラウドのマネージドなイメージを使う場合、参照先NTPサーバーは既定で適切に設定されていることが多く、手を入れなくても問題ないケースがほとんどだった。

### 自動起動・起動
```
sudo systemctl enable chronyd
sudo systemctl start chronyd
```
* インストール時点で自動起動が有効になっているディストリビューションも多いため、`systemctl list-unit-files | grep chronyd` で `enabled` かどうかを先に確認するとよい。

### 確認
* `date` で時刻が正しいことを確認する。
* `timedatectl` で時刻同期が有効かつ同期済みであることを確認する。
* 同期先のサーバーと状態の詳細
```
chronyc sources -v
chronyc tracking
```

* (参考) https://hackers-high.com/linux/easy-chrony-settings/
