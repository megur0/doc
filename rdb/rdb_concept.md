---
title: "基本概念 - RDB"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > 基本概念


## 基本概念

### SQLの規格

標準SQLはISO/IEC 9075「Database Language SQL」という国際規格に準拠しており、ANSIとISOが定めた標準SQLとなっている。

- 最新版はSQL:2016(?)


### ACID特性

トランザクションシステムの性質として定義された概念。

「守るべき概念」という説明のされ方をすることが多いが、(IMO)それぞれ「ケースバイケース」「トレードオフ」の側面が強いと考えている。

- 原子性 (Atomicity)
- 一貫性 (Consistency)
- 分離性 (Isolation)
- 永続性 (Durability)

(IMO)このうち一貫性(Consistency)だけは、他の3つと性質が異なると感じている。「あらかじめ与えられた整合性を満たすことを保証する性質」と説明されることが多いが、これはケースバイケースの話ではなく絶対に守るべきルールであり、他の3項目（トレードオフの余地があるもの）とは毛色が違う。

自分の中では、一貫性を「ある処理において何回やっても同じ結果をもたらすこと（冪等性, Idempotency）」と読み替えると理解がしっくりくる。この読み替えだと、たとえば複数のトランザクションをその実行順番によって結果が変わる（＝冪等性がない）こと（直列化異常）を許容するケースが多いことも、トレードオフの一種として説明できる。ただしこれは一般的なACIDの定義とは異なる、あくまで個人的な解釈である点に注意。


### セッション、コネクション、トランザクション

- コネクションはデータ転送用の通信路（サーバー ⇔ クライアントの通信路）
- セッションはその通信路を使っての会話（コネクションを使ったサーバー ⇔ クライアントの会話）
- トランザクションは、一連の処理全体（セッションの中で実行される一連の処理）

「`BeginTx` 〜 `Commit`（`Rollback`）」はこのうちどれに当たるかというと、トランザクションである。

- openしたコネクションがcloseされなければ、それはひとつのセッションになるため、まったく別の目的で複数のトランザクションを実行したとしても、それはひとつのセッションになる。
- コネクションの個数 ＝ セッションの個数となる。データベース側で確認できる「セッションの個数」はコネクションの数と考えてよい（実行中のトランザクション数ではない）。
- 参考: [BEGIN](https://www.postgresql.jp/docs/9.2/sql-begin.html) / [統計情報コレクタ](https://www.postgresql.jp/docs/9.2/monitoring-stats.html)

#### 1つのセッション（コネクション）が複数のトランザクションを同時に実行することはできるか

できない。

- (参考) [postgresql.org message](https://www.postgresql.org/message-id/20060330100912.277aee4f.gry%40ll.mit.edu)
- (参考) [Stack Overflow: PostgreSQL multiple transactions on the same connection](https://stackoverflow.com/questions/11620263/postgresql-multiple-transactions-on-the-same-connection)
- 例えば、Table PlusでBEGINを1つのタブで実行した際、別のタブでSQLを打つと、そのトランザクションの中で実行していることになる。
- Goの`database/sql`でも、リクエストごとに別のコネクションを利用する。


### コネクションプール

あらかじめDBへの接続をプールしておいて、接続時の負荷を軽減する仕組み。

- (参考) [コネクションプーリングとは - Qiita](https://qiita.com/ora_gonsuke777/items/0deee607922ba5dc7e94)

#### (IMO) バックエンドを複数にすることと、コネクションプール

コネクションプールはバックエンド側（例: Goのアプリケーション）で管理されるが、実装上は基本的にサーバーをリセットしない限り、プールに蓄えられているコネクションはcloseされない(?)。

仮にDBへ直接接続するバックエンドが複数存在する場合、それぞれがコネクションプールを持つことになるため、それらの上限を設けて、その合計がDB自体の最大コネクション数を超えないようにする必要がある(?)。もしくは、DBからコネクションを取得する際に上限に達していた場合の振る舞いを精査しておく必要がある(?)。


### N+1問題

あるテーブルにN件のデータがあり、別のテーブルから関連データを紐づけて取得するケースを考える。

その場合、素直にSQLを実行すると、N+1回のSQL実行が必要になるという問題。

- N件のデータを取得: 1回のSQL実行
- 関連するデータをそれぞれ検索: N回のSQL実行

解決策は主に以下の2つ。

- JOINを使う
    - 1回のSQLで実行できる。
    - ただしJOINのコストがDB側にかかる。
- Eager Loading
    - N件のデータをまず取得。
    - それに紐づくデータをIN句などを利用してまとめて取得する。
    - それをアプリケーションコード側で結合する。

(IMO) 基本的にはEager Loadingの方がアプリケーション側に処理コストを渡せるので、良い選択肢だと考えている。

- (参考) [N+1問題を徹底解説 - Qiita](https://qiita.com/muroya2355/items/d4eecbe722a8ddb2568b)


### DDL・DML・DCL・TCL

#### DDL (Data Definition Language)

テーブル、インデックス、シーケンスなどの定義を行う言語。`CREATE`、`DROP`、`ALTER`、`TRUNCATE` など。

#### DML (Data Manipulation Language)

データの検索や登録、削除といった操作を行う言語。`SELECT`、`UPDATE`、`INSERT`、`DELETE`、`EXPLAIN`、`LOCK TABLE` など。

#### DCL (Data Control Language)

`GRANT`、`REVOKE` など。

#### TCL (Transaction Control Language)

`BEGIN`、`COMMIT`、`ROLLBACK` など。


### プリペアードステートメント (prepared statement)

- SQL文の特定の箇所を変数のように後から変更できる状態（プレースホルダ）で記述された、一種の雛形（テンプレート）。
- 事前にコンパイルされ、高速に実行できる。
- 値の挿入はSQL文の解釈とは別に言語処理系によって行われるため、入力文字列の一部を誤ってSQL文の一部として解釈してしまう危険（SQLインジェクション）がない。
