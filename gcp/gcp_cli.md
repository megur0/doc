---
title: "gcloud CLI - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > gcloud CLI

* サービス別の具体的なコマンド例は[サービス別のコマンド例](./gcp_cli_examples.md)にまとめている。

## リファレンス
* https://cloud.google.com/sdk/gcloud/reference


## Google Cloud CLI(gcloud)のインストール
* https://cloud.google.com/sdk/docs/install-sdk
* 公式手順にしたがってアーカイブを配置し`install.sh`を実行する方法と、Homebrewを使う方法がある。自分は前者を使っている(IME)。
* Homebrewを使う方法
    * `brew install --cask google-cloud-sdk`
    * (参考) https://zenn.dev/phi/articles/gcloud-setup-with-homebrew-on-mac
* beta、alphaコンポーネント
    * `gcloud beta --help` / `gcloud alpha --help` を実行した際に未インストールであれば、インストールするか聞かれるのでYで進める。

### アップデート
* `gcloud components update` でアップデートする。
* ドキュメントに載っているコマンドが「存在しない」と言われる場合、SDKのバージョンが古いことが原因のケースがある(IME)。まずアップデートを試すとよい。
* バージョン確認は `gcloud --version`。


## 方針・前提事項

### デフォルトのプロジェクト、リージョン、ゾーン
* gcloudのconfigにセットしておくと、コマンド実行時にこれらを指定しなくてもよくなる。
    * configに設定されておらず、コマンドでも指定しない場合はプロンプトで聞かれる(?)。
* ただし自分としては、暗黙的にデフォルトのプロジェクトやリージョン・ゾーンでコマンドが実行されると事故が起こりそうなので、基本的に設定しない方針にしている(IMO)。都度`--project`等で明示する。
* 確認
    * `gcloud config configurations list`
    * `gcloud config list`

### gcloud auth loginで何ができるか
* gcloudのコマンドで各種の設定や作成ができるのは、`gcloud auth login`でログインしたアカウントがオーナーであり、十分な権限を持っているからである。
* 例えばロールをほとんど持たないサービスアカウントでログインした場合は、ほとんど操作できない。
* たとえばSecret Managerでのシークレット作成であれば、`roles/secretmanager.admin`のロールが付与されている必要がある。
    * https://cloud.google.com/secret-manager/docs/access-control
    * https://cloud.google.com/secret-manager/docs/creating-and-accessing-secrets


## グローバルフラグ
* `--help`
* `--project [PROJECT_ID]`
* `--account`
    * 指定しない場合は`core/account`の値がデフォルトとなる。
* 他多数。

### gcloud topic
* `gcloud topic filters` など、コマンドではなくトピック単位のヘルプが用意されている。

### --format, --filter
* `--format=flatten`と`--filter`を組み合わせる方法は使いづらい(IMO)。ドキュメントを見る限り表現力が低く、記法も分かりにくい。
    * https://cloud.google.com/sdk/gcloud/reference/topic/filters
    * (参考) https://qiita.com/okmtz/items/9d73cc2085b26cf9f208
* `--format=json`と`jq`を組み合わせるのが扱いやすい(IMO)。
    * (参考) https://stackoverflow.com/questions/73223673/filter-gcp-iam-policies-by-description


## アカウント・認証
* https://cloud.google.com/sdk/docs/authorizing
* アカウントの一覧
    * `gcloud auth list`
* gcloud CLIに対してユーザーアカウントでのアクセスを許可する
    * `gcloud auth login`
    * これを行うと、現在アクティブな構成(config)のアクティブアカウントになる。
    * 認証情報は`~/.config/gcloud/`配下に保存される。
    * ユーザーアカウントではなくサービスアカウントを使うことも可能(サーバー上でCLIを使いたい場合など)。
        * `gcloud auth login --cred-file=[KEY_FILE]`
        * `gcloud auth activate-service-account 〜` という方法もある。
* アカウントリストから削除
    * `gcloud auth revoke [ACCOUNTS ...]`
    * アカウントを省略すると、現在アクティブになっているアカウントがrevokeされる。
* ADCの認証
    * `gcloud auth application-default login`
    * `gcloud auth login`とまとめて実行したいときは `gcloud auth login --update-adc`
    * ADCそのものについては[IAM・サービスアカウント・ADC](./gcp_iam.md)を参照。
* ADCのquota project
    * `gcloud auth application-default login`した時点のカレントプロジェクトが設定される。
    * ただし、カレントプロジェクトを設定しない運用にしている場合(都度`--project`で指定する場合)、以下のwarningが出るが気にしなくてよい。
        * `WARNING: Cannot find a quota project to add to ADC`
        * (参考) https://blog.pokutuna.com/entry/gcloud-without-current-project

### Terraform等のサードパーティツール
* (参考) https://blog.pokutuna.com/entry/application-default-credentials
* `gcloud auth application-default login`でログインしておくことで、Terraform側ではこのADCの認証情報にもとづいて操作できる。


## プロジェクト
* 一覧(表示されるのは、現在のアクティブなアカウントにアクセス権限のあるもの(?))
    * `gcloud projects list`
    * Cloud Shellであれば `echo ${DEVSHELL_PROJECT_ID}` でカレントのプロジェクトIDが取れる。
* 紐づく権限を確認
    * `gcloud projects get-iam-policy [PROJECT_ID]`
* 作成
    * `gcloud projects create [PROJECT_ID] --organization=[ORGANIZATION_ID]`
* 削除
    * `gcloud projects delete [PROJECT_ID]`
* 課金アカウントが有効かの確認
    * https://cloud.google.com/billing/docs/how-to/verify-billing-enabled


## 構成(コンフィグ)
* https://cloud.google.com/sdk/docs/configurations
* 対話形式の初期化
    * `gcloud init`
    * 現在のconfigurationの初期化 / 別のconfigurationの初期化 / 新規作成 を選べる。
    * アカウント(`gcloud auth list`で確認できるもの)とプロジェクトを選択する。
* 一覧
    * `gcloud config configurations list`
* 現在アクティブなコンフィグを確認
    * `gcloud config list`
    * 例
    ```
    NAME         IS_ACTIVE  ACCOUNT            PROJECT         COMPUTE_DEFAULT_ZONE  COMPUTE_DEFAULT_REGION
    default      False
    test         True       user@example.com   example-project
    ```
* 作成
    * `gcloud config configurations create test`
    * 作成すると自動的にそのコンフィグがアクティブ状態(activate)になる。
* 切り替え
    * `gcloud config configurations activate [CONFIGURATION_NAME]`
* 削除
    * `gcloud config configurations delete [CONFIGURATION_NAME]`
* 設定
    * プロジェクト
        * `gcloud config set project [PROJECT_ID]`
        * `gcloud config get-value project`
    * アカウント
        * `gcloud config set account user@example.com`
        * これは`gcloud auth login`済みのアカウント(`gcloud auth list`で出てくるもの)のみが対象と思われる(?)。
    * リージョン・ゾーン
        * `gcloud config set compute/zone asia-east1-a`
        * `gcloud config set compute/region asia-northeast1`
* 設定解除
    * `gcloud config unset ml_engine/local_python`


## API(サービス)の有効化
* 有効なAPIを確認
    * `gcloud services list --enabled`
* 有効化・無効化
    * `gcloud services enable ml.googleapis.com`
    * `gcloud services disable ml.googleapis.com`
* フィルタ
    * `gcloud services list --filter=NAME:iap`
    * `gcloud services list --filter=NAME:compute`
* まとめて有効化する例
    ```sh
    gcloud --project [PROJECT_ID] services enable run.googleapis.com cloudbuild.googleapis.com secretmanager.googleapis.com artifactregistry.googleapis.com sqladmin.googleapis.com
    ```

### APIキーの作成
* APIキーの制限については https://cloud.google.com/docs/authentication/api-keys#api_key_restrictions
* Google Maps APIを例にした作成手順は[Google Maps API](./gcp_google_map_api.md)にまとめている。


## サービスアカウント
* 作成
    * `gcloud iam service-accounts create [SA_NAME] --display-name [DISPLAY_NAME] --description [DESCRIPTION]`
* 削除
    * `gcloud iam service-accounts delete [SA_NAME]@[PROJECT_ID].iam.gserviceaccount.com`
* キーの作成
    * `gcloud iam service-accounts keys create path/to/key.json --iam-account [SA_NAME]@[PROJECT_ID].iam.gserviceaccount.com`
    * キーの発行はなるべく避け、権限借用(impersonation)を使うほうがよい。[IAM・サービスアカウント・ADC](./gcp_iam.md)を参照。
* アクティベート
    * `gcloud auth activate-service-account [SA_NAME] --key-file [KEY_FILE]`
* サービスアカウントに権限を付与
    * `gcloud projects add-iam-policy-binding [PROJECT_ID] --member serviceAccount:[SA_NAME]@[PROJECT_ID].iam.gserviceaccount.com --role roles/storage.admin`


## 権限(ロール)の付与
* (参考) https://qiita.com/atsumjp/items/9e623edab1f7ab08fa6e
* 組織単位、フォルダ単位、プロジェクト単位、リソース単位で、ロールの対象リソースの範囲を指定できる。
    ```sh
    gcloud organizations add-iam-policy-binding ORGANIZATION --member=MEMBER --role=ROLE
    gcloud resource-manager folders add-iam-policy-binding FOLDER --member=MEMBER --role=ROLE
    gcloud projects add-iam-policy-binding PROJECT --member=MEMBER --role=ROLE
    gcloud secrets add-iam-policy-binding SECRET_NAME --member=MEMBER --role=ROLE
    ```
* GoogleアカウントをIdentity、サービスアカウントをResourceとした場合
    ```sh
    gcloud iam service-accounts add-iam-policy-binding SERVICE_ACCOUNT --member=<Google Account> --role=ROLE
    ```


## ロールの検索・確認
* `gcloud iam roles list --filter "name ~ tasks"`
* `gcloud iam roles describe roles/iam.serviceAccountUser`
* 個々の権限(permission)の説明はコマンドからは分からない。
    * API(gRPC)レベルの粒度の話であるため、APIリファレンスを読んだほうが早い(IME)。
    * https://cloud.google.com/iam/docs/reference/rest/v1/projects.serviceAccounts/list


## その他のコマンド
* VPC作成
    * `gcloud compute networks create 〜`
* サブネット作成
    * `gcloud compute networks subnets create 〜`
* ファイアウォールルール作成
    * `gcloud compute firewall-rules create 〜`
* VMインスタンス作成
    * `gcloud compute instances create 〜`
* (参考) コマンドの一覧的なまとめ https://qiita.com/komiya_____/items/4cef3b2d47ed42a7caf5


## (IME) ブランディングやOAuth 2.0クライアントの設定をgcloudから行おうとして分からなかった話
* `gcloud iam`や`gcloud iap`にoauth-brands / oauth-clientsのコマンドは存在するが、これらの`list`の出力がコンソールの表示内容と一致しなかった。
* 結局、コンソールから設定する方法に切り替えた。

```sh
# 前提:
## ブランドの設定: https://console.cloud.google.com/auth/branding
## OAuth 2.0クライアントの設定: https://console.cloud.google.com/auth/clients

# コンソール上ではクライアントが存在するにもかかわらず、結果は 0 items になる
gcloud iam oauth-clients list --location=global --project=[PROJECT_ID]

# iap.googleapis.com を有効にした上で以下を実行するとブランドは表示された
gcloud iap oauth-brands list --project=[PROJECT_ID]

# しかし以下を実行しても何も表示されない
gcloud iap oauth-clients list [上記のブランドのNameを指定] --project=[PROJECT_ID]
```
