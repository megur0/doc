---
title: "IAM・サービスアカウント・ADC - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > IAM・サービスアカウント・ADC


## IAM (Identity and Access Management)
* https://cloud.google.com/iam/docs/overview
* 誰(ID、Identity)がどのリソースに対してどのようなアクセス権(ロール)を持つかを定義する。

### プリンシパル
* リソースへのアクセスが許可されている、以下のようなもの。
    * Googleアカウント(エンドユーザー)
    * サービスアカウント(アプリケーションまたはコンピューティングワークロード)
    * Googleグループ
    * Google Workspaceアカウント
    * Cloud Identityドメイン
* 注意: 単なる「Identity」ではなく、「アクセスが許可されたIdentity」であることを指す。
    * そのため、ドキュメントに記載されている「プリンシパルとして追加」は、新しいIDを追加するという意味ではなく「アクセス権を付与する」という意味になる。
    * 例えば下記の表現では「シンクの書き込みID(これはサービスアカウントの話)をプリンシパルとして追加」とあるが、これは文脈上すでに存在するサービスアカウントに対して権限を付与する、という意味で書かれている。
        * https://cloud.google.com/logging/docs/export/configure_export_v2#dest-auth
        > Cloud Storage のエクスポート先の場合は、IAM を使用してシンクの書き込み ID をプリンシパルとして追加してから、ストレージ オブジェクト作成者のロール（roles/storage.objectCreator）をそれに付与します。
* プリンシパルは一意の識別子(通常はメールアドレス)を持つ。
* 参考
    * Principal identifiers (`allUsers`、`allAuthenticatedUsers`など)
        * https://cloud.google.com/iam/docs/principal-identifiers
    * コンビニエンス値
        * Cloud Storageがサポートしている特殊なプリンシパル。基本ロールを付与されたプリンシパルとIAMロールを付与されたプリンシパルを橋渡しする。
        * https://cloud.google.com/storage/docs/access-control/iam#identities

### ロール
* IAMでは、リソースに対するアクセス権を直接エンドユーザーに付与することはない。
* 複数の権限をロールにまとめて、認証されたプリンシパルに付与する。

### 許可ポリシー(IAMポリシー)
* どのプリンシパルにどのロールが付与されるのかを定義し、適用する。
* 各許可ポリシーはリソースに適用される。
* 認証されたプリンシパルがリソースにアクセスしようとすると、IAMはリソースの許可ポリシーをチェックし、そのアクションが許可されているかどうかを確認する。

### 利用者の権限はなるべく絞る
* (参考) https://attsun1031.github.io/blog/gcp-three-things
* とりあえず Project Editor や Project Owner をつけてしまうのは良くない。
* 組織・フォルダ・プロジェクト単位でリソースに対するロールを指定する具体例
    * (参考) https://qiita.com/atsumjp/items/9e623edab1f7ab08fa6e

### ロールや権限の検索・確認
* コンソールの「IAMと管理 > ロール」でフィルタ欄にプロパティ名やサービスのキーワードを入れると、対応するロールを検索できる。
* さらにロールの詳細を見ると保持している権限が分かるため、ここから必要な権限セットをおおよそ類推できる(IME)。
* CLIからの検索は[サービス別のコマンド例](./gcp_cli_examples.md)を参照。


## サービスアカウント
* https://cloud.google.com/iam/docs/service-account-overview
* https://cloud.google.com/iam/docs/service-accounts-create
* ユーザーではなく、アプリケーションやCompute Engineインスタンスなどのコンピューティングワークロードで使用される特別なアカウント。
* サービスアカウントは、アカウント固有のメールアドレスで識別される。
* アプリケーションをサービスアカウントとして認証する最も一般的な方法は、アプリケーションを実行しているリソースにサービスアカウントをアタッチすること。
* サービスアカウントはプリンシパル(ID)であると同時に、リソースでもある(後述の権限借用に関係する)。
    * https://cloud.google.com/iam/docs/service-account-permissions

### サービスアカウントは必要以上に使わない
* https://cloud.google.com/iam/docs/best-practices-for-using-and-managing-service-accounts
* (参考) https://attsun1031.github.io/blog/gcp-three-things
* 要するに「キーを流出させたくないので、日常の開発では`gcloud auth login`(gcloudツールなど)や`gcloud auth application-default login`(Terraform等の場合)を使いましょう」という話。

### デフォルトで作成されるサービスアカウント
* プロジェクトを作成すると、Compute Engineのデフォルトサービスアカウント(`PROJECT_NUMBER-compute@developer.gserviceaccount.com`)が作成される。
* 一覧は`gcloud --project [PROJECT_ID] iam service-accounts list`で確認できる。

```
DISPLAY NAME                     EMAIL                                                       DISABLED
myapp-cloud-run-sa               myapp-cloud-run-sa@[PROJECT_ID].iam.gserviceaccount.com     False
Default compute service account  [PROJECT_NUMBER]-compute@developer.gserviceaccount.com      False
```

### デフォルトのサービスアカウントの扱い
* (参考) https://blog.pokutuna.com/entry/application-default-credentials
* 実行環境にアタッチするサービスアカウントを明示的に指定しない場合は、デフォルトのものが使われる。
* デフォルトでは編集者(`roles/editor`)ロールという広い範囲を操作できる権限が付いている。
    * そのまま使うのはリスクがありそうだが、毎回作成するのも面倒ではある(IMO)。
* 管理の方針として良さそうなもの(IMO)
    * 小さなプロジェクトでは気にせずデフォルトのサービスアカウントのまま動かす。
    * デフォルトサービスアカウントを何かから参照したくなったら変える(別プロジェクトのIAMに追加して権限を与えたくなった、キーを外部に置きたくなった、など)。
* なお、Secret Managerのシークレットを読むには編集者(`roles/editor`)では足りず、Secret Managerのシークレットアクセサー(`roles/secretmanager.secretAccessor`)ロールが必要。

### サービスエージェント
* Google Cloudのサービスが別のサービスを呼び出すときに用いる特殊なサービスアカウント(サービスアカウントの一種)。
* コンソールのIAMページにある「Google提供のロール付与を含める」というチェックボックスをオンにすると確認できる。
* (参考) https://blog.g-gen.co.jp/entry/service-agent-explained


## サービスアカウントの権限借用(impersonation)
* https://cloud.google.com/iam/docs/service-account-overview#impersonation
* https://cloud.google.com/iam/docs/impersonating-service-accounts
* https://cloud.google.com/iam/docs/service-account-permissions
* (参考) https://qiita.com/atsumjp/items/9e623edab1f7ab08fa6e
* サービスアカウントの権限借用とは、ユーザーが有効期間の短い認証情報を使用してサービスアカウントとして認証すること。

### メリット
* (参考) https://blog.g-gen.co.jp/entry/using-google-cloud-service-account-impersonation
* ユーザー側で権限を保持する必要がなく、サービスアカウントに権限を集約できるため管理コストが下がる。
* サービスアカウントキーを発行せずに済む。
    * https://github.com/gcpug/nouhau/blob/master/general/note/destroy-service-account-key/README.md

### 関連ロール
* `roles/iam.serviceAccountUser`
    * このロールを持つプリンシパルは、対象のサービスアカウントがアクセスできるすべてのリソースに間接的にアクセスできる。
    * ただし、認証情報の作成や、gcloudの`--impersonate-service-account`フラグの使用はできない。
    * 使い分けとしては、「なりすましをする側」と「なりすましを許可する側(`roles/iam.serviceAccountTokenCreator`)」を分けたいときのもの、と理解している(?)。ただし本番作業などで両者が分かれていると、自分自身で昇格できず管理者の作業に依存するため運用は面倒になりそう(IMO)。
* `roles/iam.serviceAccountTokenCreator`
    * 認証情報の作成、gcloudの`--impersonate-service-account`フラグの使用ができる。
    * gcloudでは`CLOUDSDK_AUTH_IMPERSONATE_SERVICE_ACCOUNT={SA_EMAIL}`環境変数、または`--impersonate-service-account={SA_EMAIL}`でなりすまして操作を実行できる。
* 上記のロールは2つのリソース(なりすます側とサービスアカウント)の間の設定であるため、オーナーであってもデフォルトでは持っていない。

### 運用の具体例
* 「権限付与者」自体のサービスアカウントを用意しておき、作業者は作業時に「権限付与者」(`--impersonate-service-account`で指定)として、自身にロールを付与する。
    * これによって管理者や作業者は「権限付与」の権限を常時保持する必要がなくなり、また権限付与作業が管理者に依存しなくなる。
* (参考) https://techblog.tver.co.jp/entry/kurose/datasys-temp-permission
* (参考) 一時的に権限を付与する例: https://allabout-tech.hatenablog.com/entry/2022/07/21/174444
* コマンド例は[サービス別のコマンド例](./gcp_cli_examples.md)を参照。


## IAMの一覧確認
* コンソールでは「IAMと管理 > IAM」で確認できる。
* コマンドではプリンシパル単位で一覧するものは無いようだ(?)。ポリシー一覧であれば出せるので、これでおおよそ把握できる(ポリシー単位で表示される点に注意)。
    * `gcloud projects get-iam-policy [PROJECT_ID]`
    * (参考) 出力を加工してプリンシパル単位で表示する例: https://qiita.com/elyunim26/items/d8ee0760d1974e6720fd
* `gcloud --project [PROJECT_ID] iam 〜`と`gcloud projects 〜 [PROJECT_ID]`の2系統のコマンドがあるが、棲み分けの基準はよく分かっていない。


## ADC (Application Default Credentials)
* https://cloud.google.com/docs/authentication/application-default-credentials
* (参考) https://blog.pokutuna.com/entry/application-default-credentials
* Googleが提供するクライアントライブラリは、以下の優先順位で認証情報を解決する。ADCとは、この認証情報を解決するフローやライブラリ、または得られた認証情報のことを指す。

1. `GOOGLE_APPLICATION_CREDENTIALS`環境変数に設定されたサービスアカウントキー(のファイルパス)
2. 上記が設定されていない場合は、コードを実行しているリソースに関連付けられているサービスアカウントを使用する。
    * ローカル環境: `~/.config/gcloud/application_default_credentials.json`に配置されたOAuth2の認証情報。これは`gcloud auth application-default login`によって配置される。実行するとブラウザが起動してGoogleアカウントのログインが求められ、ログインしたユーザーの認証情報が配置される。
    * Google Cloud上の環境: 実行している環境(Cloud Run等)に紐付けられたサービスアカウント。

    ```go
    // 例(Firebase)
    app, err := firebase.NewApp(context.Background(), nil)
    // 例(Cloud Storage)
    storageClient, err := storage.NewClient(ctx)
    ```

3. サービスアカウントキーへのファイルパスをプログラムで直接渡す。

    ```go
    // 例(Firebase)
    opt := option.WithCredentialsFile("path/to/serviceAccountKey.json")
    app, err := firebase.NewApp(context.Background(), nil, opt)
    // 例(Cloud Storage)
    client, err := storage.NewClient(ctx, option.WithCredentialsFile(jsonPath))
    ```

* アプリケーションをクライアントライブラリで動作させる場合、プログラムはADCの認証情報を前提として動作する。
    * つまり自分のローカルでは動作していたのに、ADCが設定されていない環境でプログラムを動かすとエラーになる、ということが起こるので注意(IME)。
* Terraformのようなサードパーティ製CLIツールも、2の方式を使っていることが多い。
