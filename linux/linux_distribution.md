---
title: "ディストリビューション - Linux"
updated: 2026-08-29
---

[TOP(About this memo))](../README.md) > [一覧(Linux)](./README.md) > ディストリビューション

## 系統
主要なディストリビューションは、大きくRed Hat系とDebian系に分けられる。パッケージマネージャが異なるため、まずどちらの系統かを把握するのが実務上は重要になる。

### Red Hat系（パッケージマネージャは yum / dnf）
* Red Hat Enterprise Linux（RHEL、有償）
    * Red Hat社が開発した商用向けディストリビューション。サポート提供のため有償となる。
* CentOS Linux（無償）
    * かつてはRHELのソースをもとにした無償の互換ディストリビューションとして提供されていた。RHELとほぼ同じでありながら無償で利用でき、セキュリティアップデートも安定して提供されるという理由で広く使われていた。
    * **CentOS Linuxは現在すべてのバージョンがEOLを迎えている**（CentOS Linux 8は2021年12月、CentOS Linux 7は2024年6月30日にサポート終了）。新規に採用する対象ではない。
    * 後継として、RHELの上流にあたるローリングリリース版の「CentOS Stream」が提供されている。ただし性格が異なり、従来のCentOS Linuxのように「RHELの安定した無償版」を求める用途には、AlmaLinuxやRocky Linuxといった互換ディストリビューションが使われることが多い。
* Amazon Linux
    * AWSが提供するディストリビューション。Red Hat系をベースとしており、RHELやCentOSとほぼ同じように使用できる。
    * Amazon Linux 2 → Amazon Linux 2023 への移行が進んでいる（後述）。

### Debian系（パッケージマネージャは APT系コマンド）
* Ubuntu（無償）
    * デスクトップ用途・サーバー用途ともに利用者が多く、シェアが非常に高い。
    * LTS（Long Term Support）版は標準で5年間のサポートが提供される。加えてUbuntu Proに含まれるESM（Expanded Security Maintenance）とLegacyアドオンにより、最長で15年間のセキュリティメンテナンス期間が用意されている（執筆時点の情報）。
    * (?) かつては「CentOSに比べてサポート期間が短い」と言われることがあったが、CentOS LinuxのEOLとUbuntuのサポート期間延長により、現在この比較はあてはまらない。
* Debian（無償）
    * Ubuntuのベースになったディストリビューション。完全に有志コミュニティによって開発されている。

## Amazon Linux 2 と Amazon Linux の違い
* (参考) https://qiita.com/manabusakai/items/0ba9504136383f06ddd7
* CentOSと似た構成のため、Amazon Linux（初代）からAmazon Linux 2への変更点は、概ねCentOS 6から7への変更点と同じ。
* ファイルシステムが ext4 から xfs に変更。
* initデーモンが Upstart から systemd に変更。
    * `pstree` コマンドで見ると、PID 1のデーモンが init から systemd に変わっていることが分かる。
* `ec2-user` の uid と gid が 500 から 1000 に変更。
* Yumリポジトリがひとつに集約された。`ls -l /etc/yum.repos.d/` を見ると、epelリポジトリが削除されたことが分かる。
* MTA が Sendmail から Postfix に変更。

## Amazon Linux 2 と Amazon Linux 2023 の違い
* Amazon Linux 2 のサポートは2026年6月30日で終了する予定となっており、Amazon Linux 2023 への移行が推奨されている（執筆時点の情報。最新のサポート期限はAWSの公式ドキュメントで確認すること）。
    * https://docs.aws.amazon.com/linux/al2023/ug/what-is-amazon-linux.html
* 主な変更点
    * パッケージマネージャが `yum` から `dnf` に変更（`yum` はdnfへのエイリアスとして残っている）。
    * `amazon-linux-extras` が廃止された。
    * 時刻同期が `ntpd` から `chronyd` に変更。
    * ファイアウォールが `iptables-services` から `nftables` に変更。
    * デフォルトではSSHのパスワード認証が無効。
