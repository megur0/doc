---
title: "Artifact Registry・Container Registry - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > Artifact Registry・Container Registry


## docker pushの前提知識
* Docker全般については[Docker](../docker/README.md)を参照。ここではレジストリへpushする際に関係する部分のみ整理する。

### docker (image) tag
* イメージにタグ付けをする。
    * ここでいう「タグ付け」という言葉が分かりづらいので注意。
    * 「タグ付け」は既存のイメージに別名をつける、というニュアンス。コピーやリネームではなく、同じDockerイメージを指し続ける点がポイント。
    * なお、Dockerの「タグ」は`centos:latest`の`latest`のようなタグのことで、ここでいう「タグ付け」やコマンドの`tag`とは別物。
* 具体例: `docker tag httpd:test example/myimage:version1.0.test`
    * 既存のイメージ(名前: `httpd`、タグ: `test`)について、別名(名前: `example/myimage`、タグ: `version1.0.test`)として「タグ付け」する。
    * 実行後、`docker images`で確認するとタグ付けした一行が追加されている(イメージ自体がコピーされたわけではない)。
* `docker build`の`-t`(`--tag`)も同様にタグ付けを行うオプション。
    * `docker build -t example/go:latest .`
* https://docs.docker.com/reference/cli/docker/image/tag/

### docker push
* `docker login 〜` でレジストリへログインしてから `docker push [IMAGE]` を実行する。
    * AWSやGoogle Cloudでは、専用のCLIコマンドを実行することでログインする(内部的に`docker login`相当のことをしていると思われる(?))。
* このときのイメージ名は`[リポジトリURL]/[名前]:[タグ]`のように、それぞれのレジストリの命名規則にしたがっている必要がある。


## Container RegistryとArtifact Registry
* 現在はArtifact Registryを使う。Container Registryは2023年5月に非推奨となり、2025年3月18日にシャットダウンされている。
    * https://cloud.google.com/artifact-registry/docs/transition/prepare-gcr-shutdown
    * ただし、Artifact Registry上でホストされている`gcr.io`のURL(Google提供のイメージを含む)は引き続き利用できる。`gcr.io`というドメイン自体が使えなくなったわけではない点に注意。
* Artifact Registryは、Container Registryの機能を改善・拡張したもので、DockerイメージのほかにnpmやMaven、Pythonなどのパッケージ管理も統合されている。
* Container Registryと異なり、先にリポジトリを作成する必要がある。
    * `gcloud artifacts repositories create 〜` ([コマンド例](./gcp_cli_examples.md))
* ホスト名の形式
    * Container Registry: `gcr.io/` 系
    * Artifact Registry: `[リージョン]-docker.pkg.dev/` 系
* https://cloud.google.com/artifact-registry/docs/docker/copy-from-gcr


## Artifact Registryの構造
* レジストリ > リポジトリ > イメージ、という階層になっている。

```
レジストリ
└ リポジトリ
　└ イメージ
```

* https://cloud.google.com/artifact-registry/docs/transition/transition-from-gcr#compare
* リポジトリを先に作成しておかないと、pushの際にエラーになる。
* イメージURLの形式の違い
    * Container Registry(旧): `gcr.io/[プロジェクトID]/[イメージ名]:[タグ]`
    * Artifact Registry: `[リージョン]-docker.pkg.dev/[プロジェクトID]/[リポジトリ名]/[イメージ名]:[タグ]`
