---
title: "Cloud Observability(Logging・Monitoring) - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > Cloud Observability


## Google Cloud Observability
* https://cloud.google.com/products/operations
* オブザーバビリティ
    > オブザーバビリティは、環境の状態を理解するためにテレメトリー データを収集、分析する包括的なアプローチ
    * 指標
    * ログ
    * トレース
    * 他のデータ
* プロジェクトの作成時にデフォルトで有効になるサービス
    * Cloud Monitoring
    * Cloud Logging
    * Cloud Trace

### 各サービス
* Cloud Monitoring
* Cloud Logging
* Cloud Trace
    * https://cloud.google.com/trace/docs/overview
    * Google Cloudの分散トレースシステム。
    * (参考) https://zenn.dev/monicle/articles/682d406e69b5ba
* Error Reporting
* Cloud Profiler


---
## Cloud Logging

### 公式
* https://cloud.google.com/logging/docs/overview
* API
    * https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry

### 参考
* (参考) https://zenn.dev/knowledgework/articles/cloud-logging-special-payload-fields


## ログの種類
* https://cloud.google.com/logging/docs/overview#categories
* プラットフォームログ
    * Google Cloudサービスによって書き込まれるログ。
* コンポーネントログ
    * プラットフォームログに似ている。システム上で実行されるGoogle提供のソフトウェアコンポーネントによって生成されるログ。
* 監査ログ
    * Cloud Audit Logs
    * アクセスの透明性ログ(Googleのスタッフがデータにアクセスした際のログ)
* ユーザー作成のログ
* マルチクラウドログとハイブリッドクラウドログ
    * 他のクラウドプロバイダのログや、オンプレミスインフラストラクチャのログ。


## 監査ログ(Cloud Audit Logs)
* https://cloud.google.com/logging/docs/audit
    > データアクセス監査ログは、サポートチームがアカウントの問題をトラブルシューティングするのに役立ちます。このため、有効にしておくことをおすすめします。
* (参考) https://hajimenoit.com/google-cloud08/
* 管理アクティビティ監査ログ
    * リソースに対する重要な変更に関するログ。
    * 常に、ログが生成されたプロジェクトの`_Required`バケットに保存される。
* データアクセス監査ログ
    * データサイズが非常に大きくなる可能性があるため、BigQueryのデータアクセス監査ログを除き、デフォルトでは無効になっている。
    * 別の場所に転送しない限り、(`_Required`ではなく)`_Default`ログバケットに保存される。
* システムイベント監査ログ
    * システム自体が実行したアクションのログ。
    * 常に、ログが生成されたプロジェクトの`_Required`バケットに保存される。
* ポリシー拒否監査ログ
    * セキュリティポリシー違反が原因で、Google Cloudサービスがユーザーやサービスアカウントへのアクセスを拒否した場合に記録される。
    * `_Default`ログバケットに保存される。


## Cloud Loggingの料金
* https://cloud.google.com/stackdriver/pricing#logging-pricing-summary
* 以下は執筆時点の金額。変動するため必ず公式ページで確認すること。
* ログの取り込みにかかる料金
    * $0.50/GiB
    * プロジェクトあたり50GiB/月が無料枠。
    * この料金には、取り込み先バケットでの30日間の保持が含まれる。
* ログの保持にかかる料金
    * デフォルトの30日を超えて保持する場合、$0.01/GiB/月。月単位で請求される。
    * 保持期間をデフォルトの30日のままにしておけば、この料金はかからない。
    * ログバケットは通常のCloud Storageバケットよりも高額のため注意。
* 参考
    * (参考) https://blog.g-gen.co.jp/entry/cloud-logging-explained
    * (参考) https://zenn.dev/uma002/articles/03fae99ae84603


## Cloud Loggingの各プロパティ
* https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry
* InsertId
    * ログエントリに対する一意ID。特に設定しなくても自動的に割り当てられている。
    * カスタムで設定することも可能だが、業務アプリではあまりカスタムする意味はなさそう(IMO)。リクエスト単位で一意なIDは欲しいが、ログエントリ単位でのIDをカスタムする必要性は薄いため。


## 構造化ログ
* https://cloud.google.com/logging/docs/structured-logging
* Goのslogでの構造化ログ
    * (参考) https://zenn.dev/hytkgami/articles/cloud-logging-with-slog
    * (参考) https://speakerdeck.com/nownabe/go-de-cloud-logging-wo-shi-ikonasutameno-slog-huo-yong-fa

### (IME) OpenTelemetryの採用を見送った話
* Cloud TraceはOpenTelemetryを利用することができる。
* ただ、アプリサーバー性能のボトルネックのトレースは、現時点では費用対効果の観点で不要と判断した。
    * DB処理のボトルネックのほうが問題になりやすく、アプリサーバー側がボトルネックになるケースは少ないと考えたため。
    * もし必要になっても、自分でトレース情報を埋め込むほうが手軽だと考えた。
    * ライブラリの構成が複雑で好みに合わない、という点もある。
* 結果として、ログのJSONにトレース情報を埋め込みたいだけだったため、採用を見送った。
* https://opentelemetry.io/docs/languages/go/getting-started/


## ログルーター・シンク・ログバケット
* https://cloud.google.com/logging/docs/routing/overview
* ログルーター
    * Cloud LoggingはCloud Logging APIでログエントリを受信し、その過程でログエントリはログルーターを通過する。
* シンク
    * ログルーターのシンクは、Cloud Loggingがログをルーティングする方法を制御する。
    * シンクを使用すると、ログの一部またはすべてをサポートされている宛先に転送できる。
    * デフォルトで作成されるシンク
        * `_Required`
            * `_Required`バケットへ転送する。
            * 無効化や動作の変更はできない。
        * `_Default`
            * `_Default`バケットへ転送する。
            * 無効化や動作の変更ができる。
    * 転送先としてサポートされている宛先
        * Cloud Loggingバケット
        * BigQueryデータセット
        * Cloud Storageバケット
        * Pub/Subトピック
        * Splunk
        * Google Cloudプロジェクト
* ログバケット
    * ログバケットはログをリアルタイムに分析するためのバケット。
        * Cloud Loggingに保存したログはインデックスに登録されて最適化され、リアルタイムで分析できるように配信される。
        * Cloud Loggingバケットは、似た名前を持つCloud Storageバケットとは異なるストレージエンティティ。
        * 通常のCloud Storageバケットよりも高額。
    * デフォルトで生成されるログバケット
        * https://cloud.google.com/logging/docs/routing/overview#default-bucket
        * これらのバケットは削除できない。
        * `_Required`
            * 監査ログ(Audit Logs)を格納する特殊なバケット。
            * 変更や削除ができない。
            * 取り込み料金もストレージ料金もかからない。
            * 400日間保持され、保持期間は変更できない。
        * `_Default`
            * `_Required`に格納するログ以外のログは、`_Default`シンクを無効化・変更しない限りここに格納される。
            * デフォルトで30日間保持される。
            * 削除できない。
            * 保持期間を変更することが可能。ただしデフォルトの保持期間を超えた分は課金対象となる。
                * https://cloud.google.com/logging/docs/buckets#custom-retention
    * ユーザー定義のログバケット
        * 任意のプロジェクトでユーザー定義のログバケットを作成可能。
        * https://cloud.google.com/logging/docs/buckets#create_bucket


## (IMO) 良さそうなCloud Loggingの設定方針
* ログバケットの保持期間のデフォルト設定(30日)は変えない。
* Cloud Storageへ転送するようにしておく。
    * Cloud StorageはAutoclassを有効にしておく。
* 参考
    * (参考) https://zenn.dev/uma002/articles/03fae99ae84603
    * (参考) https://iga-ninja.hatenablog.com/entry/2022/04/16/153735


## Cloud Storageへの転送
* https://cloud.google.com/logging/docs/export/configure_export_v2
* (参考) https://zenn.dev/pharmax/articles/f60e0d857f2be2
* 作成したシンクには、Cloud Storageのオブジェクト作成者ロール(`roles/storage.objectCreator`)を付与する必要がある点に注意。

### 注意点: 途中から転送をはじめた場合
* 例えば、Cloud Loggingで`_Default`への保持期間を1年間にしておき、1年間運用したとする。
* その後、料金を安くするためにCloud Storageへの転送設定をしたとする。
* その際、Cloud Storageへ転送されるからといって、直後に`_Default`の保持期間を1ヶ月に変更してはいけない。
* これは、転送設定は設定を行った直後のログから有効になるため、それ以前のログは転送されないためである。
* したがって、過去のログまで含めて1年間保存する要件を満たすのならば、転送設定後さらに1年経過してから`_Default`の保持期間を1ヶ月にする、という対応が必要になる。
* 参考: 過去のログをCloud Storageへ保存する機能もある。
    * (参考) https://tech.pepabo.com/2024/03/27/glouc-logging-copy/
    * (参考) https://zenn.dev/uma002/articles/03fae99ae84603

### Cloud Storageに転送されたログの確認
* https://cloud.google.com/logging/docs/export/storage
* ログの確認は、バケットへ保存されているJSONファイルを確認することになる。
* 1時間ごとに一括してCloud Storageバケットに保存される。
* 最初のエントリが表示されるまでに2〜3時間を要する場合がある。


## ログバケットの保持期間の更新
* https://cloud.google.com/logging/docs/buckets#custom-retention
* ログバケットの保存ログバイト数に関するアラートも設定できる。
* ただ、分析する期間から考えて、デフォルトの保持期間を変更する必要性はあまりなさそう(IMO)。
* ログバケットは通常のCloud Storageバケットに比べて高額なため、デフォルトのままにしておいてCloud Storageへ転送するほうが良さそう(IMO)。


---
## Cloud Monitoring: ログベースのアラートポリシー
* Cloud Monitoringでは様々な指標に応じてポリシーを作成できる。ここではログをトリガーとした(ログベースの指標の)ポリシーの作成について記載する。

### 通知チャネルの作成
* https://cloud.google.com/monitoring/alerts/using-channels-api

### アラートポリシーの作成
* https://cloud.google.com/logging/docs/alerting/log-based-alerts#lba-edit
* `documentation`
    * `content`: ドキュメントの内容。
    * `mimeType`: 必須。有効な値は`"text/markdown"`のみ。
* `combiner`
    * 指標ベースのアラートポリシーで、複数の条件の結果を組み合わせる方法を指定する項目。
    * ログベースのアラートポリシーは単一の条件のみ指定可能で、この値は固定で`"OR"`とする必要がある。
* `alertStrategy`
    * `notificationRateLimit`
        * 通知の最低限の間隔。
        * 例えば300sとした場合は、条件を満たすログがどれだけ発生しても、前回の通知から300s以上経過するまでは次の通知が送られない(と理解している(?))。
    * `autoClose`
        * インシデントを自動的にクローズする期間。デフォルトは1週間(604800s)であり、最大値も1週間となる。
* テスト
    * https://cloud.google.com/logging/docs/alerting/log-based-alerts#example

### フィルターの検証
* コンソール上で直接ポリシーを作成すると、フィルターの入力箇所で「Preview logs」により過去のログから検索をかけてくれる。
* これを確認することで、フィルターが適切に機能しているか確認できる。
