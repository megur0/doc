---
title: "gcloud サービス別のコマンド例 - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > gcloud サービス別のコマンド例

* gcloudの基本的な使い方(インストール・認証・構成)は[gcloud CLI](./gcp_cli.md)を参照。
* 以下の例では、プロジェクトIDやサービスアカウント名などは`[PROJECT_ID]`のようなプレースホルダーで記載している。

## Secret Manager: シークレットの作成
* https://cloud.google.com/secret-manager/docs/creating-and-accessing-secrets#secretmanager-create-secret-gcloud
* (参考) https://qiita.com/toshi0607/items/fc2f106bbcd2d5ff3c7c

```sh
gcloud secrets create [SECRET_ID] --replication-policy="automatic" --data-file=[FILE_PATH]
```

* 以下の2つは同じ結果になる。
    1. `gcloud secrets create [SECRET_ID] --replication-policy="automatic" --data-file=[FILE_PATH]`
    2. `gcloud secrets create [SECRET_ID] --replication-policy="automatic"` の後に `gcloud secrets versions add [SECRET_ID] --data-file=[FILE_PATH]`


## Artifact Registry: リポジトリの作成
```sh
gcloud --project [PROJECT_ID] artifacts repositories create [REPOSITORY_NAME] \
  --repository-format=docker \
  --location=us-west2 \
  --description="example repository"

gcloud artifacts repositories list
```

* Artifact Registryの概念については[Artifact Registry・Container Registry](./gcp_artifact_registry.md)を参照。


## Cloud SQL: インスタンスの作成
* https://cloud.google.com/sql/docs/postgres/create-instance
* まずコンソールから作成を一度やってみると、主要な設定項目を把握できる(IME)。
* オプションの詳細は `gcloud sql instances create --help` を参照。

```sh
gcloud sql instances create [INSTANCE_NAME] \
    --tier=db-f1-micro \
    --no-backup \
    --database-version=POSTGRES_14 \
    --region=[REGION] \
    --root-password=[ROOT_PASSWORD]
```

* PostgreSQLの場合、`--root-password`は`postgres`ユーザーのパスワードになると思われる(?)。
* 利用可能なデータベースバージョンは `gcloud sql database versions list` で確認できる。
* マシンタイプ(`--tier` / `--cpu` / `--memory`)の考え方や、バックアップ等の各設定については[Cloud SQL](./gcp_cloud_sql.md)を参照。


## サービスアカウントの作成と権限付与(Cloud Runの例)
* https://cloud.google.com/iam/docs/service-accounts-create#iam-service-accounts-create-gcloud

作成:
```sh
gcloud --project [PROJECT_ID] iam service-accounts create [SA_NAME] \
  --description="[DESCRIPTION]" \
  --display-name="[DISPLAY_NAME]"
```

シークレットへアクセスできるロールを付与:
```sh
gcloud --project [PROJECT_ID] secrets add-iam-policy-binding [SECRET_ID] \
  --member=serviceAccount:[SA_NAME]@[PROJECT_ID].iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

Cloud SQLに接続できる権限を付与:
```sh
gcloud projects add-iam-policy-binding [PROJECT_ID] \
  --member="serviceAccount:[SA_NAME]@[PROJECT_ID].iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

### WIP: Cloud Run内のプログラムからCloud Tasksにタスクを追加したい場合
* Cloud Tasksに追加できる権限
    ```sh
    gcloud projects add-iam-policy-binding [PROJECT_ID] \
      --member=serviceAccount:[SA_EMAIL] --role=roles/cloudtasks.enqueuer
    ```
* Cloud Runを起動できる権限(タスク追加時にCloud RunのURLを指定するため必要になると思われる(?))
    ```sh
    gcloud projects add-iam-policy-binding [PROJECT_ID] \
      --member=serviceAccount:[SA_EMAIL] --role=roles/run.invoker

    gcloud iam service-accounts add-iam-policy-binding [CLOUD_RUN_SA_EMAIL] \
      --member=serviceAccount:[SA_EMAIL] --role=roles/iam.serviceAccountUser
    ```
* 参考
    * https://cloud.google.com/tasks/docs/reference-access-control
    * (参考) https://goodbyegangster.hatenablog.com/entry/2021/07/31/205847


## 一時的な権限付与(条件付きロール)の例
* (参考) https://allabout-tech.hatenablog.com/entry/2022/07/21/174444
* IAM Conditionsで有効期限を指定し、権限借用と組み合わせて一時的にロールを付与する例。

```sh
gcloud projects add-iam-policy-binding [PROJECT_ID] \
  --member='group:[GROUP_ID]' \
  --condition="expression=request.time < timestamp(\"$(date '+%Y-%m-%dT%TZ' -u -d '30 minute')\"),title=tmp_$(date '+%Y-%m-%d-%T')" \
  --role='[ROLE]' \
  --impersonate-service-account=[GRANTER_SA]@[PROJECT_ID].iam.gserviceaccount.com
```

* 権限借用の考え方は[IAM・サービスアカウント・ADC](./gcp_iam.md)を参照。


## Cloud Run: デプロイの例
サービスアカウント・シークレット・環境変数・Cloud SQLの接続を指定する例。

```sh
gcloud --project [PROJECT_ID] run deploy [SERVICE] \
  --image [IMAGE_URL] \
  --service-account [SERVICE_ACCOUNT] \
  --allow-unauthenticated \
  --set-cloudsql-instances=[PROJECT_ID]:[REGION]:[CLOUD_SQL_INSTANCE_NAME] \
  --set-env-vars=INSTANCE_CONNECTION_NAME=[PROJECT_ID]:[REGION]:[CLOUD_SQL_INSTANCE_NAME] \
  --tag=v${SHORT_SHA} \
  --set-secrets=POSTGRES_PASSWORD=[SECRET_ID]:1
```

* `INSTANCE_CONNECTION_NAME`は `gcloud sql instances describe [INSTANCE_NAME]` で確認できる。
    * プログラム内では `/cloudsql/INSTANCE_CONNECTION_NAME` をUnixソケットのパスとして指定する。
* `--set-cloudsql-instances`で指定したCloud SQLがパブリックIPパスの場合、Cloud Runは2つの方法(Unixソケット経由 / Cloud SQLコネクタを使用)でCloud SQL Auth Proxyを使用して暗号化と接続を行う。
    * この2つの方法のメリット・デメリットの違いはよく分かっていない。とりあえずUnixソケット経由にしておけば問題なさそう(IMO)。
    * プライベートIPパスの場合、アプリケーションはサーバーレスVPCアクセスを介してインスタンスに直接接続する。
    * 参考
        * https://cloud.google.com/sql/docs/postgres/connect-instance-cloud-run
        * https://cloud.google.com/sql/docs/postgres/connect-run
        * (参考) https://zenn.dev/s_edy/articles/2ba8358497731b
        * (参考) https://blog.g-gen.co.jp/entry/from-cloudrun-to-cloud-sql-with-auth-proxy

### 構成のエクスポート・更新
```sh
gcloud run services describe [SERVICE] --format export > service.yaml
gcloud run services replace service.yaml
```

* Cloud Runでのシークレットの利用については https://cloud.google.com/run/docs/configuring/secrets
