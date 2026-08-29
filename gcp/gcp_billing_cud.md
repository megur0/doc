---
title: "料金・無料枠・CUD(確約利用割引) - Google Cloud"
updated: 2026-08-27
---

[TOP(About this memo))](../README.md) > [一覧(Google Cloud)](./README.md) > 料金・無料枠・CUD

* 金額はいずれも執筆時点のものであり、変動する。実際の見積もりは必ず公式の料金ページと料金計算ツールで確認すること。

## 無料枠
* https://cloud.google.com/free


## 料金計算ツール(Pricing Calculator)
* https://cloud.google.com/products/calculator


## CUD (Committed use discounts / 確約利用割引)
* https://cloud.google.com/docs/cuds
* 一定量の利用を一定期間確約することで、割引を受けられる仕組み。
* 購入は、Google Cloudコンソールの「お支払い > 確約利用割引 > 購入」から行う。
    * https://cloud.google.com/docs/cuds-spend-based

### 費用ベース / リソースベース
* 費用ベース: 「毎月いくら払うので割引してもらう」というもの。
* リソースベース: Compute Engineのみが対象。

### 用語
* 正確な意味を理解しきれていないため、以下は文脈からの推測(?)。
* オンデマンドコミットメント
    * 割引前の使用量(料金)。
* コミットメント
    * 割引後の使用量(料金)。

### 契約内容
* 一度契約すると、期間中は使っても使わなくても請求される。
* 割引は、契約した請求先アカウントに紐づくプロジェクトの対象使用料へ適用される。
* 契約は年単位(1年、3年)。
* 契約はキャンセルできない。
* 契約開始日は契約を開始した時点。
    * 例えば1年契約で契約開始が10/15の場合、10/15〜翌年9/30となる。
* 請求は月単位で、契約した翌月から請求される。
* 割引の適用は1時間単位。
    * 1時間あたりのオンデマンドコミットメントが測定され、それに対して割引が適用される。
    * したがって1時間あたりの使用量の変動が大きい場合は、確約分を使い切れず無駄が出やすい。
        > CUD による利点を得られるリージョンのすべてのインスタンスで vCPU とメモリの 1 時間あたりの料金を計算する場合は、まず、費用を抑えることができるかどうかを考慮します。この上限を超えると、通常のオンデマンド料金で請求されます。
    * 具体例: https://cloud.google.com/sql/cud#how_to_calculate_an_hourly_on-demand_commitment

### 日割りができない点に注意
* 月の途中から契約しても、その月の過去の分には割引が適用されない。
* しかし、確約料金自体は月単位で請求される。
    * たとえば10/31に契約すると、10/1〜10/31分の確約料金が請求される。
    * しかし割引が適用されるのは10/31分のみ。
    * したがって10/31に契約すると、ほぼ1ヶ月分の料金を損することになる。
* 日付の基準はPST(太平洋標準時)/PDT(太平洋夏時間)と思われる(?)。
    * PSTはJST-17、PDTはJST-16。
    * したがって日本時間では、契約開始は1日目の夜以降にするべきだろう(IMO)。JSTの1日の朝に行うと前月扱いとなってしまう。
* (参考) https://n-s.tokyo/2022/03/gcp-cud-tips/

### 円安・円高の影響
* ドル建ての契約金額自体は為替では変動しない。したがって「円安が原因で当初決めたCUDが達成できなくなる」という事は起きない。
* もちろん、円換算の最終的な支払額は円安なら上がり、円高なら下がる。

### Cloud SQLのCUD
* https://cloud.google.com/sql/cud
* マシンタイプが変わっても適用される。
* リージョンはCUDを契約したリージョン限定。
    * 契約したリージョンに含まれるインスタンス全てに適用される。
    * つまり$1000の確約をした場合、$500と$500のインスタンスが2つあれば、確約分の割引は両方に適用される($1000を超えた分は割引適用されない)。
* 適用対象
    * 確約したリージョンのすべてのCloud SQLデータベースインスタンスのCPUとメモリ使用量。
* 適用対象外
    * 永続ディスクのスナップショット、ストレージ、IPアドレス、下り(外向き)ネットワーク、ライセンス。
    * 共有CPUマシンタイプ(`db-f1-micro`、`db-g1-small`など)。
* 割引率は執筆時点で、1年間の確約が25%、3年間の確約が52%。
* Cloud SQL自体の設定・料金は[Cloud SQL](./gcp_cloud_sql.md)を参照。

### TODO・不明点
* Cloud SQLは後からCUDを追加購入できる? -> おそらくできそう(?)。
    * (参考) Compute Engineでは追加やマージができそう。 https://medium.com/google-cloud/how-to-reduce-your-google-cloud-compute-engine-bill-by-50-with-committed-use-discounts-part-2-18239ccb70c2
    * (参考) Compute Engineで追加購入している例。 https://tech.uzabase.com/entry/2024/09/11/165942

### その他参考
* (参考) https://zenn.dev/ptna/articles/36ceb256dc32fc
* (参考) https://zenn.dev/team_zenn/articles/zenn-used-cud
* (参考) https://www.cloudzero.com/blog/gcp-cud/
