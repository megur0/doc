---
title: "トランザクション分離レベル - RDB"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > トランザクション分離レベル


## トランザクション分離レベル

### atomic（不可分性）とisolate（分離性）

ひとつのステートメントは、atomicであり、isolateである(?)。ステートメントはサブクエリを含んでいても、ひとつのステートメントとして扱われる。これはさすがにどのRDBMSでも保証されているはず。

- (参考) [Can sub-selects change in one single query in a READ COMMITTED transaction? - DBA Stack Exchange](https://dba.stackexchange.com/questions/210485/can-sub-selects-change-in-one-single-query-in-a-read-committed-transaction)

一方、`BEGIN` 〜 `COMMIT`（`ROLLBACK`）のトランザクションは、一般的にatomicではあるが、isolateではないので注意が必要。

- atomic
    - 処理はアトミックになるので、すべてが反映される、またはすべて反映されない。
- isolate
    - 一般的に、何もしなければデフォルトでは同時にトランザクションが発生しうるので注意が必要。つまり同時実行制御の考慮が必要になる。
    - 分離の程度は分離レベルによって異なる。
    - SERIALIZABLEモードの場合はすべて直列、つまり完全に分離した状態とほぼ同等になる。シンプルになる一方、並行処理ができなくなるため、性能面とのトレードオフやチューニングの検討が必要になる。


### 各異常状態（ANSI定義）

※ ファジーリードとファントムリードは、ネット上の説明が若干わかりづらい（表現に揺れがある）ため、以下は自分なりの言葉で整理している。

- ダーティリード
    - 同時に実行されている他のトランザクションが書き込んで、まだコミットしていないデータを読み込んでしまうこと。
- 反復不能読み取り（ファジーリード）
    - トランザクションが、以前読み込んだデータと異なる結果になること。更新処理に対するもの。
- ファントムリード
    - トランザクションが、以前読み込んだデータと異なる結果になること。追加処理に対するもの。
- 直列化異常
    - 複数のトランザクションを正常にコミットした結果が、それらのトランザクションを1つずつあらゆる可能な順序で実行する場合の結果と一貫性がないこと。


### 各分離レベル（ANSI定義）

- READ UNCOMMITTED
    - 上記の異常がすべて発生しうる。
    - PostgreSQLではこのモードを指定しても内部でREAD COMMITTEDとして扱われるため、実質このモードは存在しない。
- READ COMMITTED
    - ダーティリードは発生しない。
- REPEATABLE READ
    - ダーティリード、ファジーリードが発生しない。
    - PostgreSQLではファントムリードも発生しない（更新競合検査による）。
    - MySQLでもファントムリードは発生しない（ギャップロック・ネクストキーロックによる。ただし意図しないデッドロックには注意）。
- SERIALIZABLE
    - ダーティリード、ファジーリード、ファントムリード、直列化異常のいずれも発生しない。

MySQLとPostgreSQLの実装の違いは以下の通り。

- MySQLはロック制御をすべて悲観ロックで行う。
- PostgreSQLは悲観ロックをレコードロックのみに抑え、それ以外は楽観ロックで対応しようとしている。
- 両者ともMVCCを採用しているため、単なるSELECTではロックが発生しない。
- MySQLのデフォルト分離レベルはREPEATABLE READ。
- PostgreSQLのデフォルト分離レベルはREAD COMMITTED。


### A Critique of ANSI SQL Isolation Levels

- (参考) [A Critique of ANSI SQL Isolation Levels(日本語解説)](https://developer.hatenastaff.com/entry/2017/06/21/100000)
- SQLのANSI規格で与えられている分離レベルの定義について、理論的解析を行いながら批判している論文（1995年、かなり古典的）。
- 要旨
    - ANSI標準の分離レベルの定義は曖昧かつ不備があるため、それに代わる定義を提案している。この定義はロックを用いて定義される分離レベルとも対応づけができる。
    - Snapshot Isolationが、望ましい分離レベルの特徴づけになっていることを示している。

#### Snapshot Isolation

- (参考) [A Critique of ANSI SQL Isolation Levels(日本語解説)](https://developer.hatenastaff.com/entry/2017/06/21/100000)
- (参考) [Snapshot Isolationについて - Qiita](https://qiita.com/kumagi/items/1dc1a91ec007365ac694)
- Write Skew AnomalyとRead Only Anomalyが発生する。ANSI定義のREPEATABLE READでは防ぐことができない。
- 一方でPhantom Read Anomalyは発生しない。ANSI定義のREPEATABLE READでは防ぐことができないが、PostgreSQLやMySQLのREPEATABLE READでは防げる。


### Anomaly

Anomalyとは、Serializableでない実行を引き起こす異常状態パターンのことを言う。弱い分離レベルほどAnomalyが起きやすい。

- (参考) [Anomalyについて - Qiita](https://qiita.com/kumagi/items/5ef5e404546736ebac49)
- (参考) [PostgreSQLにおけるトランザクション分離レベル - Speaker Deck](https://speakerdeck.com/mpyw/postgres-niokerutoranzakusiyonfen-li-reberu?slide=35)

ANSI定義にはないAnomalyとして、以下のようなものがある。

- Read Skew
    - `T1: R(X) → T2: W(x), W(y) → T1: R(y)` のように、xとyの整合性が合わなくなる。READ COMMITTEDで発生する。
- Lost Update
    - READ COMMITTEDで発生する。実際の実装では、ファジーリード・ファントムリードを防止する（＝トランザクション開始時点での最新バージョンを見る）代わりに、ロストアップデートが発生してしまう（直前のバージョンを見ないということは、逆に直前の変更が失われてしまうデメリットがある）。
        - MySQLではLocking Read/Writeで一貫性読み取りを行わないことで対応。
        - PostgreSQLでは、最後に検査を行って検知することで対応。
        - (参考) [ロストアップデートについて - Qiita](https://qiita.com/shohei1913/items/%E3%83%AD%E3%82%B9%E3%83%88%E3%82%A2%E3%83%83%E3%83%97%E3%83%87%E3%83%BC%E3%83%88)
- Write Skew
    - READ COMMITTED、Snapshot Isolationで起きる。REPEATABLE READでは起きない。
    - ざっくり言うと、直列であれば自分の書き込みが終わってから他のトランザクションが読み取るはずが、中途半端なタイミングで読み取りされることで直列と異なる結果になる現象。
        - 直列でない例: `T1: R(x) → T2: R(y) → T1: W(y) → T2: W(x)`
        - 直列な例: `T1: R(x) → T1: W(y) → T2: R(y) → T2: W(x)`
    - 具体例: `T1: y = x + 1`、`T2: x = y + 1`、初期値 `x = y = 0` とすると、直列でない場合は結果が `x = y = 1` になってしまうが、直列な場合は `x = 2, y = 1` になる。
- Read Only（Observe Skew）
    - よくわかっていない(TODO)。
- Cursor Lost Update
    - よくわかっていない(TODO)。MySQLでは起こらない(?)。
    - (参考) [What is and how to produce cursor lost update in MySQL - Stack Overflow](https://stackoverflow.com/questions/73985547/what-is-and-how-to-produce-cursor-lost-update-in-mysql)

MySQLで実験している記事もある。REPEATABLE READでLost Updateが起きているが、Locking READ/WRITEを使えば起きない、とのこと。

- (参考) [MySQLでのロストアップデート検証記事](https://tombo2.hatenablog.com/entry/2017/12/11/141156)


### MVCCと通常のselect

- 過去のコミット済みバージョン（スナップショット）を保存しておく手法。
- 各処理において最新のバージョンを取得する（Consistent Read）。トランザクション開始前に取得するか、各処理の都度、直前の最新バージョンを取得するかは分離レベルによる。
- MVCCを使っている＝ファントムリードやファジーリードが防がれるわけではない。
- MVCCを使っている＝ダーティリードは防がれる、と解釈するのはよい(IMO)。
- メリット: 行ロックをすることなく、ダーティリードを回避できる。

#### MVCC（MultiVersion Concurrency Control：多版型同時実行制御）

- (参考) [PostgreSQL: 同時実行性の制御](https://www.postgresql.jp/document/7.2/user/mvcc.html)
- (参考) [MVCC and GC in PostgreSQL](https://masahikosawada.github.io/2021/12/22/MVCC-and-GC-in-PostgreSQL/)
- (参考) [MVCCについて](https://shallow1729.hatenablog.com/entry/2021/05/17/212613)
- (参考) [Heroku: PostgreSQL Concurrency](https://devcenter.heroku.com/ja/articles/postgresql-concurrency)

#### いつスナップショットを生成するか（PostgreSQL）

古いバージョンのドキュメント（9.4系(?)）ではトランザクションの開始前にスナップショットを取得すると書かれていたが、9.5以降のドキュメントからはトランザクション内の最初の読み取り時にスナップショットを取得すると説明されている(?)。

- (参考) [PostgreSQL 9.1 Documentation: Transaction Isolation](https://www.postgresql.org/docs/9.1/transaction-iso.html)

#### いつスナップショットを生成するか（MySQL）

MySQLではトランザクション内でロック無しSELECTが実行された段階でスナップショットが生成される。ただし、そのスナップショットが違うテーブル・SQLに対してであっても、ロック無しSELECTであれば全て同一のスナップショットが適用されるのかは未確認(TODO)。

- (参考) [MySQLのREPEATABLE READについて](https://techblog.kayac.com/repeatable_read.html)


### CSR（Conflict Serializable）、2PL（2 Phase Lock）、S2PL、SS2PL、C2PL

このあたりの理論はかなり難解で、要点整理はできていない(TODO)。

- (参考) [CSR・2PLについて (1)](https://yunkt.hatenablog.com/entry/2018/09/24/115206)
- (参考) [CSR・2PLについて (2)](https://yunkt.hatenablog.com/entry/2018/09/26/000608)
- (参考) [CSR・2PLについて (3) - Qiita](https://qiita.com/kumagi/items/d3c671ddd1aa5648dd91)


### 分離レベルの無難なプラクティス（WIP）

(IMO) READ COMMITEDが良い塩梅だと思っている。REPEATABLE READにする積極的な理由はあまりない（消極的な理由もないが）。単にREAD COMMITED + Locking READ/WRITEのやり方に慣れているのであれば、MySQLでもPostgreSQLでもいずれにせよロストアップデートは対応可能、という理解。

- (参考) [PostgreSQLにおけるトランザクション分離レベルの選び方 - Zenn](https://zenn.dev/mpyw/articles/rdb-transaction-isolations)
- (参考) [PostgreSQLにおけるトランザクション分離レベル - Speaker Deck](https://speakerdeck.com/mpyw/postgres-niokerutoranzakusiyonfen-li-reberu)

> 特に SERIALIZABLE についてはチューニングの知識も求められるため，インフラも含めるとかえって学習コストが上がってしまう懸念がある。無理をして使うよりは，素直に READ COMMITTED で Locking Read を用いるほうが汎用性は高い。

補足: 「Locking Read」はREAD COMMITTEDだとファントムリードが起きるが、「勧告的ロック」（[ロック概論](./rdb_lock.md)参照）を使うことでファントムリードに対応できる。

方針として、以下のような形が挙げられる。

- 分離レベルはREAD COMMITEDにする。
- Locking Readによって明示的に行へロックを掛ける。
- Locking Readには`NOWAIT`をつける。つまり、競合した場合は待機せずに即エラーにする（リトライまたは失敗にさせる）。
- ただし、取得していない行に関してはREAD COMMITEDだとロックされない。その箇所においてアプリとして許容できないのであれば、アドバイザリーロック（勧告的ロック）を使う。

#### PostgreSQL

デフォルトのREAD COMMITTEDのまま使うのが無難。REPEATABLE READやSERIALIZABLEにすることで楽観的制御の恩恵を受けたり、統一的にシリアルにすることができるが、性能チューニング等の独自の学習コストがある。活かせるユースケースもそれほど多くないことを考えると、デフォルトのままが無難(IME)。

#### MySQL

デフォルトのREPEATABLE READではなく、READ COMMITTEDにするのが無難(IME)。REPEATABLE READにはギャップロックを使っているため、意図しないロックを発生させるおそれがあるという課題がある。

なお、Locking Read/Writeは一貫性読み取りではない。

- (参考) [InnoDBのRepeatable ReadとLocking Read](http://nippondanji.blogspot.com/2013/12/innodbrepeatable-readlocking-read.html?m=1) — ロック無しリードとの一貫性を犠牲にして、ロストアップデート対策ができるようにしている(?)。PostgreSQLの場合はロック無しリード以外も一貫性読み取りとなっているが、競合検査によってロストアップデートを検知する仕組みになっている。
- MySQL公式ドキュメントには「同じトランザクション内で複数のプレーン (非ロック) SELECT ステートメントを発行すると、これらの SELECT ステートメントも互いに一貫性が保たれます」とあり、逆説的に「それ以外は一貫性がない」ことを示唆していると解釈できる(?)。
    - (参考) [MySQL 8.0 リファレンスマニュアル: InnoDB のトランザクション分離レベル](https://dev.mysql.com/doc/refman/8.0/ja/innodb-transaction-isolation-levels.html)
    - (参考) [MySQL 8.0 リファレンスマニュアル: InnoDB の一貫性のない読み取り](https://dev.mysql.com/doc/refman/8.0/ja/innodb-consistent-read.html)
- (参考) [MySQLの一貫性読み取りについて - Qiita](https://qiita.com/shohei1913/items/52f50ae1f75ffec2696b)
