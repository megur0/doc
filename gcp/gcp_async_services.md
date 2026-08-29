---
title: "非同期・イベント駆動(Pub/Sub・Cloud Tasks・Scheduler・Workflows・Batch) - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > 非同期・イベント駆動


## イベント駆動とat least once
* 基本的にアプリケーションは冪等に作る必要がある。
* (参考) https://qiita.com/y-wat/items/7d0d3f102f35bfada488
* 各サービスの重複実行の可能性
    * Cloud Schedulerは「まれに、同じジョブの複数のインスタンスがリクエストされる可能性があります」とされており、重複実行され得る。
    * Cloud StorageからPub/Subへの通知は複数回になる可能性がある。
    * Cloud Run functionsをPub/Subトリガーで構築している場合は複数回起動する可能性がある(Pub/SubでCloud Runを呼んでいる場合も同様と思われる(?))。
    * Pub/Subはexactly onceの設定ができるが、レイテンシが低下するというデメリットがある。
    * Cloud Tasksは「99.999%以上のタスクが1回だけ実行される」と記載があるものの、「完全に重複排除はできない」という注意書きがある。


## Pub/Sub
* https://cloud.google.com/pubsub/docs/subscriber#push_pull

### Pub/SubとCloud Runの連携
* トピックを作成する。
* Cloud RunのURLへpushするサブスクリプションを、そのトピックに対して作成する。
* トピックに対してメッセージをpublishすることでCloud Runが起動する。
* https://cloud.google.com/run/docs/tutorials/pubsub

### Pub/SubとCloud Tasksの違い
* Cloud Tasksは、タスク作成時に実行側のエンドポイントを指定するため、実行側との依存が強い。またタスク単位の状況確認・削除が可能で、二重実行防止やリトライを細かく設定できる。
* 一方Pub/Subは、メッセージ作成側が実行側を指定しないため依存が弱い。1つのメッセージの状況確認や削除はできず、リトライもCloud Tasksほど細かく設定できない。
* 使い分けの考え方
    * Pub/Sub: 処理された結果とWebアプリ自体の結びつきが弱い、サービス連携に向いている。
    * Cloud Tasks: Webアプリから非同期でバックエンドのAPIを叩いてデータ更新するような、機能的に強く結びついたサービス連携に向いている。
* (参考) https://recruit.gmo.jp/engineer/jisedai/blog/gcp-tasks-pub-sub/
* push型とpull型の使い分けについては理解が浅い(TODO)。


## Cloud Tasks
* https://cloud.google.com/tasks/docs/dual-overview
* フルマネージドなメッセージキューサービス。大量のタスクの分散実行や配信管理、リトライ処理を行うことが可能。ディスパッチ間隔なども設定できる。
* キューを作成し、プログラムやコンソールからキューへタスクを追加する。タスクを実行するのはハンドラ(ワーカー)側。
* キューに乗せる際に実行するタイミングを指定することで、スケジューリング(計画的な配信)ができる。30日先まで指定可能。
* 重複排除ができる(99.99%)。
* デフォルトのタイムアウト期限は10分、最大30分以内にHTTPレスポンスコード(200〜299)をCloud Tasksサービスに返す必要がある。
* 参考
    * https://cloud.google.com/tasks/docs/creating-http-target-tasks
    * (参考) https://zenn.dev/nananaoto/articles/bd1584c77e46f128a41a
    * (参考) https://zenn.dev/google_cloud_jp/articles/e35fbe793efb5b
    * (参考) 流量調整に使う例 https://zenn.dev/waddy/articles/rails-enqueue-cloud-tasks-parallel

### タスク作成はプログラムから行う必要がある
* Cloud Tasksのタスク作成は、基本的にクライアントライブラリを使ってアプリ内のプログラムから呼び出す必要がある。
    * Web APIのエンドポイントも一応あるが、gRPCをそのまま扱うのは現実的ではない(IMO)。
* そのため、たとえばCloud SchedulerからCloud Tasksへ登録したい場合は「Scheduler → functions → Tasks → 実行したいもの」という構成になる(?)。
* ただし、実行したいものがCloud Run functionsやCloud Runのようにオートスケールしてリソースの心配が無いものであれば、「Scheduler → 実行したいもの」で全く問題ない(IMO)。
    * EC2やGCEのようにスケールしない環境が実行先であれば、Cloud Tasksのようにキューを間に入れたほうが良い。

### エミュレーター
* ローカルでテストするための公式のエミュレーターは無いようだ(?)。
* サードパーティ製のツールはある。
    * https://github.com/aertje/cloud-tasks-emulator


## Cloud Scheduler
* 「HTTPリクエストの実行」「Pub/Subのトピックへのメッセージ送信」の2つのうちから実行内容を選ぶ。
* cronのように実行時刻を設定できる。
* Cloud Runのエンドポイントを呼ぶことで、シンプルにスケジュール処理を実現できる(IMO)。


## Workflows
* 定義した順序でサービスを実行するサービス。
* Cloud RunやCloud Run functionsでホストされているカスタムサービス、BigQueryなどのGoogle Cloudサービス、任意のHTTPベースのAPIを組み合わせることができる。
* メリットは、ワークフローの流れや依存関係が可視化されること、個々のサービスの成功・失敗・再実行をコントロールできること(IMO)。
* ワークフローのスケジュール実行はCloud Schedulerを使うことで実現できる。この場合、Cloud SchedulerがWorkflows APIにリクエストを送信する。
* https://cloud.google.com/workflows/docs/tutorials/execute-cloud-run-jobs
* (参考) https://www.m3tech.blog/entry/2023/02/27/110000
* (参考) https://tech.rhythm-corp.com/schedule-workflows-using-cloud-workflows/
* (参考) Scheduler + Workflows + Batchの構成例 https://recruit.gmo.jp/engineer/jisedai/blog/gcp-batch/


## Cloud Composer
* ワークフローオーケストレーションサービス(Apache Airflowのマネージドサービス)。
* スケジューリングだけでなく、バッチ処理のパイプラインの作成やモニタリングも行える。
* 一般的な規模のサービスにとってはやや過剰な感はある(IMO)。


## Cloud Batch
* Cloud Run jobsと同じようにコンテナを指定して実行が可能。またコンテナを順次実行できる。
* Cloud Run jobsのようにCloud StorageやFilestoreが使えるほか、GCEのPersistent Disk、ローカルSSDも使える。
* 実行時間やインスタンスタイプの制限が無いため、Cloud Run jobsでは手に負えない重い処理を行う場合はBatchを使うのが良さそう(IMO)。
* Cloud Run jobsとの使い分けは正直悩ましいところ(IMO)。
* Cloud Schedulerと繋ぐためにはWorkflowsを経由する必要がありそう(?)。
* リージョンの対応状況は変わるため、利用前に公式ドキュメントで確認すること(以前は東京リージョンが利用できなかった)。
* (参考) https://zenn.dev/google_cloud_jp/articles/c99697707e3b2c
* (参考) https://blog.g-gen.co.jp/entry/large-csv-processing-cloud-batch


## Eventarc
* Googleのサービス、SaaS、独自のアプリからイベントを非同期に配信できるサーバーレスなサービス。
* https://cloud.google.com/eventarc/docs/overview
