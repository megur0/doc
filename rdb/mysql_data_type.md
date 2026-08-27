---
title: "データ型 - RDB(MySQL)"
updated: 2026-08-25
---

[TOP(About this memo))](../README.md) > [一覧(RDB)](./README.md) > データ型（MySQL）


## データ型（MySQL）

### 小数点の計算

- floatは誤差が出てしまう。
- DECIMAL型は、小数点以下を10進数で処理するので誤差が出ない。


### varcharとtextの違い

- (参考) [varcharとtextの違い](http://lxyuma.hatenablog.com/entry/2015/08/15/131309)
- textはポインタなので、パフォーマンスはvarcharの方が高い。varcharは初期値と最大文字長を指定できる。varcharもtextも最大64KB(?)。
- どちらにせよ、InnoDBは8KBの壁と言われる行サイズの制限にひっかかる可能性がある。その場合は、まずは行フォーマットをCOMPRESSEDかDYNAMICにする。
- そのため、結論としてはvarcharを使うのが一択と言えそう(IMO)。ただし、MEDIUMTEXTやLONGTEXTのようにサイズ自体が大きくなる場合はtextを使う。


### TIMESTAMPとDATETIMEの違い

- TIMESTAMPの値は、保存時にはタイムゾーンからUTCへ、読み出し時にはUTCからタイムゾーンへと変換される。
- これに対しDATETIMEはタイムゾーンの影響を受けることなく文字列として認識・保存される。読み出し時にも文字列認識により、そのままの状態で出力される。
- DATETIME: 格納できる値の範囲が広いが、その分容量が必要。
- TIMESTAMP: インデックス、UTC変換、タイムゾーン対応など、機能性を求めるならこちら。
- (参考) [MySQL DATETIME vs TIMESTAMP](https://www.codeproject.com/Tips/1215635/MySQL-DATETIME-vs-TIMESTAMP)
