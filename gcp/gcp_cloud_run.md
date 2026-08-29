---
title: "Cloud Run - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > Cloud Run

* デプロイのコマンド例は[サービス別のコマンド例](./gcp_cli_examples.md)を参照。

## Cloud RunとCloud Run functionsの使い分け
* Cloud Run functionsは、2024年8月にCloud Functionsから改称されたもの。第2世代がCloud Run functions、第1世代がCloud Run functions(第1世代)となった。基盤はCloud Runに統合されている。
* Cloud Run functionsはエンドポイント1つ単位、Cloud RunはAPIサーバーとして複数のエンドポイントを持てる、という違いがある。
* サーバーを動かすならCloud Run、Google Cloud上のリソースを連携させるちょっとしたコードならfunctions、という棲み分け(IMO)。
* https://cloud.google.com/blog/ja/products/serverless/cloud-run-vs-cloud-functions-for-serverless


## Cloud Run contract
* Cloud Runサービス、Cloud Runジョブを作成する上でのルール。全部目を通しておくべき内容(IMO)。
* https://cloud.google.com/run/docs/container-contract
* Linux 64ビット用にコンパイルする必要がある。
    * GoであればGOARCHはamd64でよい。
* ジョブが正常に完了したときに、コンテナは終了コード0で終了する必要がある。


## 既知の問題
* 一部のリージョンからの呼び出し時に、カスタムドメインでリクエストレイテンシが増加する
* 最大インスタンス数が3以下の場合に可用性が低下する
* Cloud Runのジョブタスクが再試行されたと誤認されることがある
* 他
* https://cloud.google.com/run/docs/issues


## コンテナのライフサイクル
* (公式記事) https://cloud.google.com/blog/ja/products/serverless/lifecycle-container-cloud-run
* デプロイ時にArtifact Registryからイメージを取得し、内部ストレージへコピーする。
    * したがって、一度デプロイしたイメージを誤ってArtifact Registryから削除しても動作は継続する。
* 起動時は内部ストレージのイメージから取得する。
* リクエストを処理していないときはアイドル状態になる。
* アイドル状態ではCPUはほぼゼロとなり課金されない。
* アイドル状態ではバックグラウンドタスクの実行は保証されない。
    * 確実に実行したい場合はCloud Tasksを使う。
* アイドル状態からは、トラフィックの状況に応じて随時シャットダウンされる。
* リクエスト時にサービング状態・アイドル状態のコンテナが無い場合はコールドスタートになる。
* アイドル状態によってコールドスタートは軽減されるが、アイドル状態はいつでもシャットダウンし得るので注意。
* さらにコールドスタートを軽減したい場合は最小インスタンスを構成する。
* アプリケーションはSIGTERMシグナルのハンドリングを実装しておくことでシャットダウンに備える(猶予は10秒)。
* アプリが異常終了するとアイドル状態でなくてもコンテナはシャットダウンする。
* メモリ不足でもコンテナはシャットダウンする。
* メモリ上限のデフォルトは512MiBで、変更可能(?)。


## 自動スケール
* https://cloud.google.com/run/docs/about-instance-autoscaling
* リビジョンがトラフィックを受信しない場合、デフォルトではインスタンス数がゼロにスケールインされる。
    * 最小インスタンス数を設定することも可能(当然、料金は上がる)。
* スケーリングは以下の要因の影響を受け、オートスケーラーが5秒ごとに評価する。
    * 既存インスタンスの1分間の平均CPU使用率(スケジュール設定されたインスタンスのCPU使用率を60%に維持するため)。
    * 1分間でのリクエストの最大同時実行数と比較した現在の同時実行数。
        * 最大同時実行数はデフォルトが80で変更可能。
    * インスタンスの最大数の設定(デフォルトは100で変更可能)。
    * インスタンスの最小数の設定(デフォルトは0で変更可能)。


## タイムアウト
* リクエストのタイムアウトはデフォルトで5分(300秒)。最大60分(3600秒)まで延ばせる。
* タイムアウトすると接続が閉じられ、504が返る。
* https://cloud.google.com/run/docs/configuring/request-timeout


## サービスのアクセスを無効化する
* 直接シャットダウンすることはできない。
* ただし「アクセスさせない」ことは可能。
* 一般公開している場合、「Cloud Run起動元」に`allUsers`が設定されている。
* この起動元を削除することで、サービスへのアクセスを無効化できる。
* https://cloud.google.com/run/docs/managing/services


## サービスの再起動
* 明示的な再起動という操作は無い。


## モニタリング・ロギング
* Cloud Monitoring
    * Cloud RunはCloud Monitoringに自動的に統合される(自分で連携の設定をする必要はない)。
    * やることとしてはアラート追加、指標追加など。
* Cloud Logging
    * Cloud Runには2種類のログがあり、Cloud Loggingに自動的に送信される。
        * リクエストログ(サービスのみ): Cloud Runサービスに送信されたリクエストのログ。自動的に作成される。
        * コンテナログ(サービスとジョブ): コンテナインスタンス(通常は自分のコード)から出力されたログ。
            * 標準出力・標準エラーに出していれば自動的に書き込まれる。
            * `/var/log`ディレクトリにあるすべてのファイル
            * syslog(`/dev/log`)
            * Cloud Loggingクライアントライブラリを使用して作成されたログ
    * コンソールで見ることができるほか、gcloudのコマンドからも確認可能(サービスのみ)。
    * https://cloud.google.com/run/docs/logging
    * 設計方針については[Cloud Observability](./gcp_observability.md)を参照。
* Cloud Trace
    * Cloud Runは自動的にレイテンシデータをCloud Traceへ送信している。
    * https://cloud.google.com/trace/docs/overview#configurations_with_automatic_tracing
* Cloud Audit Logs
    * 監査ログ。Google Cloudリソース内で「誰が、いつ、どこで、何をしたか」を確認できる。
* Error Reporting
    * Cloud RunはError Reportingと自動的に統合される。
    * stdout、stderrなどのログに送信されるすべての例外に加え、「メモリ上限超過」「利用できるインスタンスがありません」のサービスエラーが記録される。
    * エラーが原因でプロセスがクラッシュした場合は、新しいコンテナインスタンスの起動時にコールドスタートが発生する。


## サービスアカウント
* Cloud Runのすべてのリビジョンがサービスアカウントにリンクされている。
* このサービスアカウントは、Google Cloud APIでの認証を行うためにGoogle Cloudクライアントライブラリによって自動的に使用される。
    * Google Cloud APIの例として、Cloud Storage、Firestore、Cloud SQL、Pub/Sub、Cloud Tasksなど。
    * つまり、Cloud Run上のプログラムからGoogle CloudのAPIを使う際に、このリンクしているサービスアカウントが使われるということ。
* サービスアカウントの指定がない場合、Cloud RunはすべてのGoogle Cloud APIに対して幅広い権限を持つデフォルトのサービスアカウントにリビジョンをリンクする。
* https://cloud.google.com/run/docs/securing/service-identity


## Cloud SQLとの接続
* プロジェクトのCloud SQL Admin APIを有効にする。
* Secret Managerから取得した接続情報でCloud SQLへ接続する。
* Cloud SQL Auth Proxyを使う。
    * Cloud SQLインスタンスのパブリックIPを使用しつつ、TLS暗号化による安全な接続を実現できる。
    * データベースの接続元をIAMで制御できるため、Cloud SQL Auth Proxyを使用し、かつCloud SQLインスタンスにアクセスするIAM権限を持っているクライアントに接続元を制限できる。
    * つまり、パブリックIPのCloud SQLを安全に使える。
    * プライベートIPだとVPCを構成する必要があるため、そこが手間になる。
    * Cloud SQL Auth Proxyを使えば、ローカルからCloud SQLに接続することも可能。
    * 詳細は[Cloud SQL](./gcp_cloud_sql.md)を参照。


## 環境変数・シークレット
* シークレットの利用方法は、ボリュームとしてマウントする方法と、環境変数として読み込む方法の2つがある。自分は後者を使いたい(IMO)。
* 現在のCloud Runの設定情報はYAMLとしてエクスポートできる。`production_service.yaml`のようにコード管理しておくとよい(当然だが、YAMLには環境変数の値は入るものの、シークレットはキー情報のみが入る)。
* https://cloud.google.com/run/docs/configuring/secrets#yaml
* https://cloud.google.com/run/docs/configuring/environment-variables#yaml


## デプロイ
* `gcloud run deploy [SERVICE] --image [IMAGE_URL]`
* すでにデプロイ済みの場合は、新しいバージョンとして新しいリビジョンサフィックスが自動的に割り当てられる(自分で指定も可能)。
    * https://cloud.google.com/run/docs/deploying#command-line
* `gcloud run deploy`は`--image`をつけない場合はソースコードからデプロイしてくれるが、ビルドを完全にカスタマイズすることはできない。デモンストレーション以外では使う機会は少なそう(IMO)。
    * https://cloud.google.com/run/docs/deploying-source-code
* Cloud Buildを使う場合
    * Cloud BuildはCloud Run、App Engine、GKE、Cloud Run functionsなどへのデプロイに対応している。
    * https://cloud.google.com/build/docs/deploy-containerized-application-cloud-run
    * 権限まわりの注意点は[Cloud Build](./gcp_cloud_build.md)を参照。


## ベースイメージの選択

### distroless
* 公式
    * https://github.com/GoogleContainerTools/distroless
* Googleが提供している、必要最小限の依存のみが含まれるDebianベースのコンテナイメージ(執筆時点ではDebian 12ベース)。
* イメージのサイズが非常に小さく、aptやシェルさえも含まれていない。
    * 最小の`gcr.io/distroless/static-debian12`は2MiB程度。
* `:debug`タグをつけることでシェル等を使うことができる。
    * nonrootと一緒に使うときは`:debug-nonroot`。
* staticとbaseの違い
    * staticは最軽量。Goなど静的リンクしたバイナリ向け。
    * libc、libssl、opensslなどを使いたい場合はbaseを選択する。
    * (参考) https://zenn.dev/yoshii0110/articles/21ddb58c6f6bfa
* `:nonroot`によってnonrootのイメージを使うことができる。
    * `:nonroot`を使わなくても、Dockerfileの`USER nonroot`によって同様のことができる。
    * (参考) https://cohalz.co/entry/2021/12/11/000000

### Alpineについて
* libcに一般的な互換性が不足している。
    * Ruby、Python、Node.jsなどでNativeモジュールをバンドルしているアプリケーションの場合、パフォーマンスの劣化や互換性の問題に当たる場合があるとのこと。
    * ただしGoは`CGO_ENABLED=0`にすればNativeモジュールに依存しないので問題ない。
* Alpineの代わりに、Googleが出しているセキュアで軽量なdistrolessを使ったほうが良さそう(IMO)。distrolessのstaticはAlpineよりサイズが小さい。
* (参考) https://blog.inductor.me/entry/alpine-not-recommended
* (参考) https://zenn.dev/jrsyo/articles/e42de409e62f5d


## Cloud Run上でのFirebase Admin SDKの実行について
* Identity Platformを有効にしなくても、Admin SDKのIDトークン検証は動作する。Firebaseプロジェクトを作成していなくても動作するのは少し不思議に感じた(IME)。
    * そもそもFirebaseプロジェクトを作成してAuthenticationを使っているプロジェクトでも、Identity Platformは有効化されていない。
* FirebaseプロジェクトとGoogle Cloudプロジェクトは1対1の関係になる。
    * 既存のGoogle CloudプロジェクトIDでFirebaseプロジェクトを作成することはできるが、すでにFirebaseプロジェクトを作成済みのプロジェクトIDを使うことはできない(2つ目を作成することはできない)。
* Admin SDKでは、初期化処理においてCloud Runに紐づくサービスアカウントを自動的に検出する。
    * https://firebase.google.com/docs/admin/setup#initialize-sdk
* 違うプロジェクトで作成したトークンを送ると、下記のようにプロジェクトが違うというエラーをライブラリが返してくれる。

```
level=info msg="token verify failed. ID token has invalid 'aud' (audience) claim; expected \"[EXPECTED_PROJECT_ID]\" but got \"[ACTUAL_PROJECT_ID]\"; make sure the ID token comes from the same Firebase project as the credential used to authenticate this SDK; see https://firebase.google.com/docs/auth/admin/verify-id-tokens for details on how to retrieve a valid ID token"
```


## プログラム側で必要なこと
* Cloud Run上のFirebaseのクレデンシャル
    * Cloud Run上でプログラムを動かすと、Firebase Admin SDKは自動的にFirebaseを認識してくれるのでクレデンシャルの指定は不要。
    * Google Cloudプロジェクトは1つのFirebaseプロジェクトしか持てない。Identity Platformも同様(なお、Identity Platformを有効にするとFirebaseも包含されて有効になる)。
    * https://firebase.google.com/docs/admin/setup#initialize-sdk
* 環境変数`PORT`で指定されたポートをリッスンさせる必要がある。
    * https://cloud.google.com/run/docs/configuring/containers
* DBのホストにCloud SQLのUnixソケットパスを使う。
    * `/cloudsql/INSTANCE_CONNECTION_NAME`
    * これは特にプログラムの修正は不要だった(IME)。すでにPostgreSQL接続時に指定しているDBのhostの環境変数に、Unixソケットパスを渡せばよい。


## クライアントIPアドレスの取得
* Cloud Runは`X-Forwarded-For`ヘッダーにクライアントのIPアドレスをセットする。
* https://cloud.google.com/load-balancing/docs/https#x-forwarded-for_header
* ヘッダーの解釈を誤ると詐称を許すことになるため注意。
    * (参考) https://layer77.net/2020/03/09/a-very-common-mistake-when-parsing-the-http-x-forward-for-header/
    * (参考) https://blog.cateiru.com/entry/2024/02/19/233045


## その他の参考
* (参考) Makefileでまとめた例 https://zenn.dev/tatsuyasusukida/articles/cloud-run-go
* Cloud Run向けDockerfileの参考
    * https://github.com/GoogleCloudPlatform/cloud-run-samples/tree/main/helloworld-shell
    * (参考) https://zenn.dev/kaito2/articles/e8576b43ff522a
    * https://docs.docker.com/build/building/multi-stage/


## 比較的新しい機能(執筆時点)
* マルチコンテナ(サイドカー)
    * 1インスタンスあたり最大10コンテナまで、同じネットワーク名前空間とインメモリボリュームを共有して実行できる。
    * Nginxをリバースプロキシとして動かす、OpenTelemetry Collectorを同居させる、といった構成が可能。
* GPUのサポート
    * NVIDIA L4等のGPUをサービス・ジョブ・ワーカープールで利用できる。
* ワーカープール(Worker pools)
    * 長時間のバックグラウンド処理、バッチ、プル型のキュー消費のための機能。永続的なインスタンスを維持してキューから処理を引き取る。
* https://cloud.google.com/run/docs/release-notes


---
## ジョブ(Cloud Run jobs)

### 概要
* Cloud Runでバッチ処理などを実行するための機能。
* サービスと異なりHTTPリクエストに依らず、複数のTaskを組み合わせることで60分以上の実行や、明示的な並列処理を行うことが可能。
* Cloud Buildで頑張ってワークフローを組まずに、Workflowsと連携させて管理できる。
* ジョブ自体には外部からのHTTPリクエストを受けるエンドポイントは存在しないが、Google CloudのAPIを通じてジョブの実行を外部から呼び出すことが可能(`roles/run.invoker`の権限が必要)。
* 呼び出し方法は「コンソールで実行」「gcloudコマンドで実行」「APIを通じて実行」がある。
    * (参考) https://medium.com/google-cloud-jp/cloud-run-jobs-c963a7143367
* Cloud Schedulerと組み合わせて、定期的なバッチ処理を行うのに使える。
    * (参考) https://recruit.gmo.jp/engineer/jisedai/blog/cloud-run-jobs/
* サービスと同様、VPC内のリソースへアクセスするにはVPCコネクタを使えばよい(Cloud SQLのプライベートIPを使う場合など)。
* 1つのジョブに対してN個のタスクという構成になる。ジョブ全体のタイムアウトは無いが、タスクのタイムアウト上限は60分。
    * 処理が60分を超えるなら、分割・並列実行するか、Cloud Batchを検討する。
* シリアルに処理をつなぐ場合はWorkflowsと連携できる。
* ストレージは、Filestoreを使うならVPCコネクタが必要。Cloud StorageならFUSEを使って共有ディスクのようにマウントして利用できる。

### 再試行とチェックポイントのベストプラクティス
* 処理に冪等性をもたせる。チェックポイントを設定する。
* 関連: [非同期・イベント駆動](./gcp_async_services.md)

### Cloud Run jobs / Cloud Run functions / Cloud Batchの使い分け
* Cloud Run functionsは、わざわざコンテナを経由したくない処理で使えそう(?)。たとえばSchedulerからCloud Tasksに追加したいときなど。
* 重い処理や実行時間・インスタンスタイプの制限を超える処理はCloud Batchを検討する。
    * (参考) https://blog.g-gen.co.jp/entry/large-csv-processing-cloud-batch


---
## サービス(Cloud Run services)

### リビジョン、タグの使い方
* https://cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration#command-line
* https://cloud.google.com/run/docs/tutorials/configure-deployment-previews

#### アクセスURL
* 基本のURLはリビジョンに関係なく固定。ただしタグ付けすることで、特定のリビジョンに専用URLを設定してテストできる。
* URLには2つの形式がある。
    * Deterministic URL: `https://[TAG---][サービス名]-[プロジェクト番号].[リージョン].run.app`
        * サービスを作成する前にURLを予測できるため、サービス間通信などで扱いやすい。
        * DNSセグメント(サービス名 + プロジェクト番号 + タグ)が63文字以下の場合にのみ利用できる。
    * Non-deterministic URL: `https://[TAG---][サービス名]-[ランダムなハッシュ].a.run.app`
        * 2つ目のフィールドがランダムなハッシュのため、デプロイ前にURLを予測することはできない。デプロイ後はURLは安定している。
    * 従来のURLも引き続き利用できるため既存への影響はないが、今後は前者を利用していくほうがよい(IMO)。
    * https://cloud.google.com/run/docs/triggering/https-request
    * (参考) https://blog.g-gen.co.jp/entry/cloud-run-deterministic-url

#### トラフィックの割当て
* 最新のリビジョンにトラフィックを割り当てる(デフォルトはこの設定になっているはず)。
    ```sh
    gcloud run services update-traffic [SERVICE] --to-latest
    # または
    gcloud run services update-traffic [SERVICE] --to-revisions LATEST=100
    ```
    * 注意点として、この状態だと新しいリビジョンを作成するたびに、トラフィックがその新しいリビジョンに再割当てされる。
* 特定のリビジョンにトラフィックを割り当てる。
    ```sh
    gcloud run services update-traffic [SERVICE] --to-revisions [REVISION]=100
    ```
    * 以降、トラフィックの割当てを更新しない限り、ずっとこのリビジョンのみに割り当てられる。
    * `100`の部分を`50`にすると、既存の割当てが50%、新規の割当てが50%といった配分になる。
* 特定のタグにトラフィックを割り当てる(リビジョンと同様の考え方)。
    ```sh
    gcloud run services update-traffic [SERVICE] --to-tags [TAG]=100
    ```
* 複数のリビジョンを指定してトラフィックを分割する。
    ```sh
    gcloud run services update-traffic [SERVICE] --to-revisions rev-00005-red=25,rev-00001-bod=25,rev-00002-nan=50
    ```
    * トラフィックが100%を超えたり、既存の割当てを含めて100%に到達しない指定をするとエラーになるはず(?)。

#### タグ付けと--no-traffic
```sh
gcloud run deploy [SERVICE] --image [IMAGE_URL] --no-traffic --tag [TAG]
```

* 以下のようなURLでテスト可能になる。
    * `https://[TAG]---[サービス名]-[プロジェクト番号].[リージョン].run.app`
* デプロイの際に既存と同じタグを指定すると、既存のリビジョンから新しいリビジョンの方へタグが付け替えられる。
* `--no-traffic`は、今回のリビジョンにトラフィックが割り当てられることを避けるために使うもの。
    * 実際にやっていることは「トラフィックが常に最新のリビジョンに割り当てられる挙動を、特定のリビジョン(現在の最新のリビジョン)に固定するように変更する」というもの。前者の挙動のままだと、今回デプロイしたものへトラフィックが割り当てられてしまうため。
    * したがって、すでにトラフィックが特定のタグやリビジョンのみに割り当てられている場合は、このオプションは特に意味を持たないはず(?)。
* タグを削除する。
    ```sh
    gcloud run services update-traffic [SERVICE] --remove-tags [TAG]
    ```

#### サービスとリビジョンの確認・削除
```sh
gcloud run services list
gcloud run services describe [SERVICE]
gcloud run revisions list --service [SERVICE]
gcloud run revisions describe [REVISION]
gcloud run revisions delete [REVISION]
gcloud run services delete [SERVICE]
```

#### (?) コンソールの「最新リビジョンのURL」
* コンソールにはこの項目があり、最新のリビジョンに対してタグ付けができるように見える。
* 新しいリビジョンを追加した後に見ると「現在、この URL は別のリビジョンに割り当てられています。このフォームを保存すると、URL の宛先が代わりにこのリビジョンになります。」と表示されるが、「保存」を押しても表示が変わらなかった(IME)。挙動がよく分からない。

### --quiet
* 対話的なプロンプトを出さないようにするオプション。


### YAMLで構成をexport / replaceする方法
```sh
gcloud run services describe [SERVICE] --format export > service.yaml
gcloud run services replace service.yaml
```

* https://cloud.google.com/run/docs/configuring/containers#yaml
* ただ、そもそもdeployコマンドで都度アップデートできるうえ、それをCloud BuildのYAMLで管理しておけばよいので、この方法で構成管理をすることはないと思う(IMO)。


### ドメイン
* Cloud Runはすべてのhttpリクエストをhttpsにリダイレクトする。
    * https://cloud.google.com/run/docs/triggering/https-request

#### ドメインの選択肢
* 自動生成されたURLを使う
    * デフォルトで発行されるURLは、そのサービスを削除しない限り安定したURLなので、用途によっては十分(IMO)。
        > サービスを最初にデプロイしたときに提供される、安定した自動割り当て URL を Cloud Run の一般公開 URL として使用できます。
    * ただしサービスを削除する必要が出てきた場合や、Google Cloudから離れる場合に、完全にGoogle Cloudに依存したURLになるため移行が面倒。
    * もちろんhttpsになっており、TLS 1.3をサポートしている。
* Cloud Runドメインマッピングを使う
    * https://cloud.google.com/run/docs/mapping-custom-domains
    * Googleマネージドな証明書を自動で発行・更新してくれるため手軽。
    * ただし執筆時点でもプレビュー段階のままであり、レイテンシの問題から本番環境向けには推奨されていない。
    * 対応リージョンも限られている。
    * 東京リージョンでカスタムドメインだと遅くなる問題が知られている。
        * https://cloud.google.com/run/docs/issues
* Firebase Hostingを使用してカスタムドメインをマッピングする
    * 公式で紹介されている方法だが、この方法を採用している情報があまり見つからない(IME)。レイテンシ等のデメリットは未調査。
* ロードバランサを使う
    * 東京リージョンで遅くなる問題は発生しない。
    * Cloud CDNやCloud Armorとも連携しやすい。
    * 既存のロードバランサが無くても、Cloud Runの機能でロードバランサを新規生成して紐付けることができる。
        * https://cloud.google.com/run/docs/integrate/custom-domain-load-balancer
    * 料金はかかるが(月20ドル程度〜。執筆時点)、金額を許容できるなら最善の選択肢だと思う(IMO)。公式でも推奨されている。
