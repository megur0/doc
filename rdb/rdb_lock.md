---
title: "ロック概論 - RDB"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > ロック概論


## ロック概論

### 楽観ロックと悲観ロック

- (参考) [排他制御 - TERASOLUNA](http://terasolunaorg.github.io/guideline/current/ja/ArchitectureInDetail/DataAccessDetail/ExclusionControl.html) — かなり詳しく書かれている記事。
- 楽観ロック
    - データそのものに対してロックは行わずに、更新対象のデータが取得時と同じ状態であることを確認してから更新することで、データの整合性を保証する手法。
- 悲観ロック
    - 更新対象のデータを取得する際にロックをかけることで、他のトランザクションから更新されないようにする手法。


### 共有ロック、排他ロック

- 共有（SHARE）ロック同士は競合しない。
- 共有ロックと排他（EXCLUSIVE、占有）ロックは競合する。
- (参考) [MySQLのロックについて](https://www.wakuwakubank.com/posts/201-mysql-lock/)
- (参考) [MySQLロック関連メモ](https://taiga.hatenadiary.com/entry/2018/02/12/170109)

#### MySQLとPostgreSQLのスタンスの違い

MySQLでもPostgreSQLでも、トランザクション中のUPDATE、DELETE、およびロック付きSELECTにはロックがかかる。両者ともロックが競合したら待機する（`NOWAIT`をつけた場合は即時エラーを返す）。

ただし、REPEATABLE READレベル以上でのアプローチが異なる。

- MySQLは、ロック取得時に取得するデータが、トランザクションの最初の読み取り時のバージョンではなく「その時点の最新のバージョン」になる。
    - これはロストアップデートをさせないアプローチ。
    - REPEATABLE READレベルであるにもかかわらずこの挙動をしているため、分離レベルを一部落としていると解釈することもできる。すなわち、プレーンSELECTの「一貫性読み取り」との整合性がずれていることを意味する。
- PostgreSQLは、ロック取得時に取得するデータがトランザクションの最初の読み取り時のバージョンになる。
    - ロストアップデートは競合検査によって行い、検知された場合はエラーになる。

(IMO) PostgreSQLは楽観的制御に重きを置き、MySQLは悲観的制御に重きを置いている、という整理がしっくりくる。


### アドバイザリーロック（勧告的ロック）

明確な定義は見つからなかったが、(IMO)「アプリケーション側でコントロールできることに置いたロック」といった解釈をしている。

例えば、PostgreSQLで用意されている `pg_advisory_lock` 関数は、任意のbigintを渡してロックとして扱うことができる。

- (参考) [PostgreSQL: 明示的ロック機能](https://www.postgresql.jp/docs/9.4/explicit-locking.html)

> PostgreSQLは、アプリケーション独自の意味を持つロックを生成する手法を提供します。これは、その使用に関してシステムによる制限がないこと、つまり、正しい使用に関してはアプリケーションが責任を持つことから勧告的ロックと呼ばれます。
