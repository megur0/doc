---
title: "Cloud Build - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > Cloud Build

* イメージの保存先であるArtifact Registryについては[Artifact Registry・Container Registry](./gcp_artifact_registry.md)を参照。

## ビルド結果の表示
* `gcloud builds list`
* `gcloud builds describe [BUILD_ID]`
* https://cloud.google.com/build/docs/view-build-results


## ignoreファイル
* `.gcloudignore`を使う。
* https://cloud.google.com/sdk/gcloud/reference/topic/gcloudignore


## gcloud builds submit
```
gcloud builds submit [[SOURCE] --no-source] [--region=REGION] (省略)
    [--config=CONFIG; default="cloudbuild.yaml"
     | --pack=[builder=BUILDER],[env=ENV],[image=IMAGE]
     | --tag=TAG, -t TAG] [GCLOUD_WIDE_FLAG ...]
```

### 動作
* 指定したソースコード内のファイルが圧縮されてCloud Storageにアップロードされる。
    * `.`を指定すると現在のディレクトリがすべてアップロードされる。
    * `.gcloudignore`で対象外のファイルを指定可能。
    * トップレベルのアップロードディレクトリに`.gcloudignore`がなく`.gitignore`が存在する場合は、`.gitignore`で指定されるファイルに基づくGit互換の`.gcloudignore`がgcloud CLIによって生成される。
        * https://cloud.google.com/build/docs/running-builds/submit-build-via-cli-api
    * `--no-source`を指定した場合は、アップロードするものが何もないことを示す。
    * ソースコードの場所を指定しなかった場合は`.`がデフォルトになる(検証済み。`--help`には記載がない)。
    * Cloud Storageバケットに保存済みのアーカイブされたソースコードを使用してもよい。
* アップロード先のストレージ
    * プロジェクトで初めて`gcloud builds submit`を実行すると、Cloud Buildがそのプロジェクトに`[PROJECT_ID]_cloudbuild`という名前のCloud Storageバケットを作成する。
    * Cloud Buildはこのバケット内のコンテンツを自動的には削除しないので注意。
        * (参考) https://zenn.dev/ibaraki/articles/d524d404fffdb5
    * ビルドで使用しなくなったオブジェクトを削除するには、バケットでライフサイクル構成を設定するか、手動でオブジェクトを削除する。
* 以降の処理は`--region`で指定したロケーションのホスト環境上で行われ、Cloud Storageにアップロードされたファイルが入力として使用される。
    > デフォルトでは、Cloud Build でビルドを実行すると、ビルドは公共のインターネットにアクセスできる安全なホスト環境で実行されます。各ビルドは独自のワーカー上で実行され、他のワークロードから分離されます。
    * https://cloud.google.com/build/docs/overview#default_pools_and_private_pools
* `--tag`が指定されている場合は、イメージのビルドを行い、指定された名前でタグ付けしてリポジトリにpushする。
    * このときに使用されるビルダーは`gcr.io/cloud-builders/docker`のはず(?)。
* `--config`が指定されている場合は、指定された構成ファイルに基づいて処理が行われる。
* `--pack`が指定されている場合は、buildpacksを使ってビルド・pushする。
* 実行ログはCloud Buildの履歴から確認できる。

### 例) cloudbuild.yamlの内容にもとづいてビルド・pushする
```sh
gcloud builds submit --config cloudbuild.yaml [SOURCE_DIRECTORY]
```

* `name`でクラウドビルダーを指定できる。たとえば以下のようなもの。
    * `gcr.io/cloud-builders/docker`
    * `gcr.io/cloud-builders/gcloud`
    * `gcr.io/cloud-builders/kubectl`
* https://cloud.google.com/build/docs/build-push-docker-image
* https://cloud.google.com/build/docs/deploy-containerized-application-cloud-run

### 例) Dockerfileを使ってビルド・pushする
```sh
gcloud builds submit --tag [IMAGE_URL] [SOURCE_DIRECTORY]
```

* `IMAGE_URL`は`[ホスト名]/[プロジェクトID]/[リポジトリ名]/[イメージ名]:[タグ]`である必要がある。
* https://cloud.google.com/build/docs/building/build-containers

### 例) ソースのみでビルド・pushする(buildpacks)
```sh
gcloud builds submit --pack image=[IMAGE_URL]
```

### 番外編) ローカルのdockerコマンドでビルドしてレジストリへpushする
* Cloud Buildを使わない方法。ローカルのイメージが増え、PCの性能に依存する。

```sh
docker build . --tag [IMAGE_URL]
gcloud auth configure-docker   # 初回のみ。認証設定
docker push [IMAGE_URL]
```

### 番外編) ローカルでpackを使ってpushする
* `pack`コマンドのインストールが必要。

```sh
gcloud auth configure-docker
pack build --publish [IMAGE_URL]
```

### buildpacksについて
* 使われるベースイメージが定期的に脆弱性スキャンされるマネージドなものであるため、buildpacksを使ったほうが良い場合がある(?)。
    * (参考) https://codezine.jp/article/detail/13038
* ローカルでpackビルドして動作確認する
    * `pack build --builder=gcr.io/buildpacks/builder [NAME]`
    * これでGoogleのビルダーでpackされたときの動作確認は可能。
    * ただ、いちいちビルドして確認するのは面倒なので、Dockerfileを作って`docker compose up`で確認したほうが効率は良い(IMO)。
    * TODO: ローカルエミュレータを使うより良い方法がありそう。要調査。
        * https://cloud.google.com/run/docs/testing/local

### (IME) gcloud builds submitが内部で何を行っているのか調べた話
* ドキュメントにはAPIで実行する方法も書かれているが、「storageSource」がAPIによって生成されるのか、APIを呼ぶ前に自分でアップロードする必要があるのかは説明から読み取れなかった。
    * https://cloud.google.com/build/docs/running-builds/submit-build-via-cli-api
* 実際に呼ばれているAPIは`POST https://cloudbuild.googleapis.com/v1/projects/{projectId}/builds`であり、リファレンスを見る限り、ソースコードは手動でアップロード済みである前提でこのAPIをコールする必要がありそう。
    * https://cloud.google.com/build/docs/api/reference/rest/v1/projects.builds#Build.Source
* そのため`gcloud builds submit`は、まず指定されたソースコードをストレージにアップロードし、その後上記のAPIを呼び出しているのではないか、という推測に至った(?)。
* `--tag`が付いている場合は、以下のような`steps`を渡しているはず。この`args`の中の`.`は、`gcloud builds submit`で指定する`.`とは関係なく、Cloud Buildが`gcr.io/cloud-builders/docker`を実行する際のcontextになる。

```json
"steps": [{
    "name": "gcr.io/cloud-builders/docker",
    "args": ["build", "-t", "gcr.io/$PROJECT_ID/my-image", "."]
}]
```

* contextの話については後述の検証で正しいことが確認できた。内部で呼び出しているAPIそのものは、ログからはトレースできなかった。

### (IME) 検証: gcloud builds submitで指定するディレクトリの意味
公式のクイックスタート(https://cloud.google.com/build/docs/build-push-docker-image )を、以下の構成で試した。`test`ディレクトリの中には`quickstart.sh`を入れていない。

```
Dockerfile
quickstart.sh
test
└ Dockerfile
```

```sh
gcloud projects create [PROJECT_ID] --organization=[ORGANIZATION_ID]
gcloud --project [PROJECT_ID] services enable cloudbuild.googleapis.com artifactregistry.googleapis.com
gcloud --project [PROJECT_ID] artifacts repositories create quickstart-docker-repo \
  --repository-format=docker --location=us-west2 --description="Docker repository"

# (1) ソースコードの場所を指定しない
gcloud --project [PROJECT_ID] builds submit --region=us-west2 \
  --tag us-west2-docker.pkg.dev/[PROJECT_ID]/quickstart-docker-repo/quickstart-image:tag1

# (2) ソースコードの場所として ./test を指定する
gcloud --project [PROJECT_ID] builds submit --region=us-west2 \
  --tag us-west2-docker.pkg.dev/[PROJECT_ID]/quickstart-docker-repo/quickstart-image:tag2 ./test

gcloud projects delete [PROJECT_ID]
```

* 結果
    * (1)は成功した。ソースコードの場所を指定していないが、`.`がソースコードとしてCloud Storageへアップロードされた。実行時に`Uploading tarball of [.] to [gs://[PROJECT_ID]_cloudbuild/source/...]`というメッセージが出ていた。
    * (2)は`./test`の中身がアップロードされ、ビルド時に`quickstart.sh`がないためエラーとなった。`test`の中に`quickstart.sh`を入れて再実行したところ成功した。
* 結論
    * ソースコードのディレクトリを指定しない場合は`.`が指定される(`--help`に記載がなかったのでフィードバックした)。
    * `--tag`を指定している場合、docker実行時のcontextは`.`になる。これはソースコードのディレクトリの指定とは無関係。Dockerがビルド時に入力とするソースコードはCloud Storageから取得されるデータであり、その中身は「指定したディレクトリ」ではなく「ディレクトリの中身」が入るため、`.`以外をcontextにするとエラーになるはず。
* なお、Cloud Loggingで確認したところ`google.devtools.cloudbuild.v1.CloudBuild.CreateBuild`のAPIが記録されていた。RPC定義の`post`は https://cloud.google.com/build/docs/api/reference/rest/v1/projects.builds/create と一致している。ただし、ログ上ではリクエストbodyの中身が分からないため、`args`としてcontextの`.`を指定していることまでは確認できなかった。
    * https://github.com/googleapis/googleapis/blob/master/google/devtools/cloudbuild/v1/cloudbuild.proto


## Cloud Buildの構成ファイル
* ビルド構成ファイルの仕様
    * https://cloud.google.com/build/docs/build-config-file-schema

### 置換変数(substitution)
* `COMMIT_SHA`や`SHORT_SHA`は、トリガーで呼ばれるビルド(GitHub連携のCIなど)を対象にデフォルトで用意されているもの。
* 手動実行時はデフォルトで空文字列が入っている。`--substitutions`で指定すれば上書きできる。
* https://cloud.google.com/build/docs/configuring-builds/substitute-variable-values

### ユーザー定義のsubstitution
* ビルドフローの中で指定している置換変数に対して、`substitutions`(置換変数と値のmap)に該当の置換変数が存在しない場合はエラーになる。
    * このmapは`substitutions`フィールドで指定しても、`--substitutions`で指定してもよい。
* 逆に、`substitutions`フィールドや`--substitutions`で指定した置換変数がフローの中に存在しない場合もエラーになる。
* 以下をつけると、上記のいずれのエラーも発生しなくなる(その後、ビルドフロー実行時にコマンドで異常が発生すればそこで失敗となる)。

```yaml
options:
    substitution_option: 'ALLOW_LOOSE'
```

### imagesフィールド
* ビルドフローの中でpushしても、`images`フィールドで指定してもレジストリへのpushは実現できる。後者だとビルド結果にイメージが表示される。
* ビルドフローでpushしつつ、ビルド結果に表示したい場合は`images`を併用することも可能。
* https://cloud.google.com/build/docs/build-config-file-schema#images

### その他の設定
* `--cache-from`を使った最適化
    * https://cloud.google.com/build/docs/optimize-builds/speeding-up-builds#using_a_cached_docker_image
* `waitFor`を使った並行処理
    * (参考) https://zenn.dev/google_cloud_jp/articles/streamlit-04-cicd
* マルチステージビルドのキャッシュ
    * (参考) https://qiita.com/bigplants/items/73f4e16b840207f09e81

### (?) imagesフィールドのリトライ回数について
* `steps`で失敗したときは特にリトライ処理はされない。
* しかし`images`フィールドによるpushは失敗した際に10回リトライを行った後、以下のようなメッセージが出力される。

```
ERROR: failed to push because we ran out of retries.
〜 retry budget exhausted (10 attempts): step exited with non-zero status: 1
```

* なぜこのpushの動作だけリトライ回数が設定されているのか、またこのリトライ回数をどう制御するのかは分からなかった。gcloudのコマンドにリトライ回数を調整するパラメータは見当たらなかった。

### (?) YAML内での引数の書き方
* これは通るが、
    ```yaml
    - "--tag"
    - "test0000"
    ```
* これは通らない。
    ```yaml
    - "--tag test0000"
    ```
    ```
    ERROR: (gcloud.run.deploy) unrecognized arguments:
    ```
* 1行単位で1つの引数として認識されているため、と思われる(?)。


## Cloud Buildのサービスアカウント
* https://cloud.google.com/build/docs/cloud-build-service-account

### デフォルトのサービスアカウントの変更(2024年)
* 2024年5〜6月に、新規プロジェクトにおけるCloud Buildのデフォルトサービスアカウントの挙動が変更された。
    * 変更前: Cloud Build専用のサービスアカウント(`PROJECT_NUMBER@cloudbuild.gserviceaccount.com`。現在は「Cloud Buildレガシーサービスアカウント」と呼ばれる)
    * 変更後: Compute Engineのデフォルトサービスアカウント(`PROJECT_NUMBER-compute@developer.gserviceaccount.com`)
* この変更より前に最初のビルドを実行済みの既存プロジェクトは、従来どおりレガシーサービスアカウントが使われる。
* そのため、プロジェクトの作成時期によってデフォルトのサービスアカウントが異なる点に注意。
* https://cloud.google.com/build/docs/cloud-build-service-account-updates

### デフォルトの権限
* デフォルトでは、同じプロジェクトのリポジトリからのアップロード・ダウンロードの権限が付与される(別プロジェクトのリポジトリであればArtifact Registry書き込みロールが必要)。
* サービスアカウントにロールを付与する方法は3通り。
    * コンソールの「Cloud Build > 設定 > サービスアカウント」
        * ざっくりした割当てになるが、簡単な用途ならこれで十分(IMO)。
    * コンソールのIAMページでCloud Buildのサービスアカウントにロールを付与・削除する。
        * https://cloud.google.com/build/docs/securing-builds/configure-access-for-cloud-build-service-account
    * gcloudのコマンドで対象のサービスアカウントに付与する。

### ユーザー指定のサービスアカウント
* Cloud Buildでユーザー指定のサービスアカウントを使うのは割と手間がかかる(IME)。可能であればデフォルトのサービスアカウントをそのまま使いたい。
* https://cloud.google.com/build/docs/securing-builds/configure-user-specified-service-accounts


## Cloud BuildからCloud Runをデプロイする際のroles/iam.serviceAccountUser
* Cloud BuildでCloud Runをデプロイする際は、Cloud Runのサービスアカウントに紐づく`roles/iam.serviceAccountUser`をCloud Buildのサービスアカウントが保持している必要がある。
    * これはCloud BuildがCloud RunのサービスアカウントのIDとしてコンテナを実行する必要があるため。
* 保持していない場合は下記のようなエラーになる。

```
ERROR: (gcloud.beta.run.jobs.deploy) User [[PROJECT_NUMBER]@cloudbuild.gserviceaccount.com] does not have permission to access namespaces instance [[PROJECT_ID]] (or it may not exist): The caller does not have permission
```

* 参考
    * https://cloud.google.com/build/docs/deploying-builds/deploy-cloud-run#continuous-iam
    * https://cloud.google.com/run/docs/securing/service-identity

### 付与のしかた
(1) プロジェクト全体の`roles/iam.serviceAccountUser`を与える。範囲が広いので本番ではやらないほうがよい。
```sh
gcloud projects add-iam-policy-binding [PROJECT_ID] \
  --member=serviceAccount:[CLOUD_BUILD_SA_EMAIL] \
  --role=roles/iam.serviceAccountUser
```

(2) Cloud Runのサービスアカウントに紐づく`roles/iam.serviceAccountUser`を与える。こちらのほうが最小限で望ましい。

ユーザー指定のCloud Runのサービスアカウントの場合:
```sh
gcloud iam service-accounts add-iam-policy-binding \
  [SA_NAME]@[PROJECT_ID].iam.gserviceaccount.com \
  --member="serviceAccount:[CLOUD_BUILD_SA_EMAIL]" \
  --role="roles/iam.serviceAccountUser"
```

デフォルトのCloud Runのサービスアカウントの場合:
```sh
gcloud iam service-accounts add-iam-policy-binding \
  [PROJECT_NUMBER]-compute@developer.gserviceaccount.com \
  --member="serviceAccount:[CLOUD_BUILD_SA_EMAIL]" \
  --role="roles/iam.serviceAccountUser"
```
