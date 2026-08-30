---
title: "リソース監視(CPU・メモリ) - Linux"
updated: 2026-08-29
---

[TOP(About this memo))](../README.md) > [一覧(Linux)](./README.md) > リソース監視(CPU・メモリ)

プロセスそのものの操作については[プロセス・ジョブ](./linux_process.md)、ディスクの使用量については[ディスク・ファイルシステム](./linux_disk.md)を参照。

## top
### macOS
* 実行中に `o` を押し、さらに `cpu` や `mem` と入力すると並び替えができる。

### Linux
* `Shift-p` … CPU使用率順
    * `m` … メモリ使用量順
    * `n` … PID順
    * `t` … 実行時間順
* `f` を押すと表示する項目を変更できる。
* CPU使用率の行の読み方
    * `us`（ユーザープロセス）と `sy`（カーネル）の合計が、おおまかな使用率にあたる。
    * より正確には `id`（idle）を100から引いた値を見る。`wa`（I/O待ち）が大きい場合はディスクがボトルネックになっている可能性がある。
* CPU使用率が100%を超えることがある
    * 複数のコアを搭載したプロセッサ、または複数のプロセッサを使用している場合、プロセス単位の使用率はコア数×100%まで表示される。
* 出力例
    ```
    top - 14:49:31 up 407 days, 15:49,  0 users,  load average: 5.19, 5.20, 5.27
    Tasks: 332 total,   1 running, 331 sleeping,   0 stopped,   0 zombie
    %Cpu(s): 31.0 us,  1.9 sy,  0.0 ni, 66.9 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
    KiB Mem : 32945628 total,   611656 free,  3485824 used, 28848148 buff/cache
    KiB Swap:  1003516 total,   972920 free,    30596 used. 28554076 avail Mem
    ```
    * CPU使用率は `100 - 66.9 = 33.1%`。
    * メモリは `total - avail Mem` がおおよその実使用量にあたる。`used` だけを見るとキャッシュが考慮されず実態とずれる。

## free
* メモリ使用量を表示する。
* `-m` や `-g` で単位を指定する。`-h` で読みやすい単位にする。
* 見るべき列
    * 現在のprocps-ngでは `total / used / free / shared / buff/cache / available` という列が表示される。**`available` を見るのが基本**。これはアプリケーションが新たに利用可能なメモリの推定値で、解放可能なキャッシュを含んでいる。
    * 古い記事にある `-/+ buffers/cache` の行は、`available` 列の導入にともなって廃止された（procps-ng 3.3.10前後、RHELでは7.1以降）。役割としては、この行の2つ目の値が現在の `available` に相当する。
* システムは使い続けるとページキャッシュとバッファキャッシュが増えていくため、`free` が小さい（`buff/cache` が大きい）こと自体は問題ではない。キャッシュは必要になれば解放される。

## vmstat
* CPU・メモリ・スワップ・I/Oの状況をまとめて、時系列で確認できる。
* `vmstat 1` のように間隔（秒）を指定すると、継続的に出力される。負荷の傾向を見るにはこの使い方が便利。
* `si` / `so`（スワップイン・スワップアウト）が継続的に発生している場合、メモリが不足している。
* (参考) https://www.pressmantech.com/tech/3993

## /proc から確認する
* `cat /proc/meminfo` … メモリの詳細な内訳
* `cat /proc/stat` … CPUごとの累積時間、コンテキストスイッチ数など
* `cat /proc/cpuinfo` … CPUの情報

### メモリ使用量の計算方法について
* (IMO) 監視ツールの中には、使用量を `MemTotal - MemFree` で計算しているものがある。これはキャッシュを「使用中」として数えてしまうため、実態より大きく見える。ページキャッシュは必要になれば解放されるので、アプリケーションから見た実質的な使用量とは言えない。
* `MemTotal - MemAvailable` を使用量とみなす方が実態に近い。これは `free` コマンドの `available` 列に対応する。
* `MemAvailable` はカーネルが算出している推定値で、`MemFree` に加えて解放可能なページキャッシュやスラブの一部を含む。`MemFree + Inactive(file) + Active(file)` を単純に足した値とは一致しない。
* (参考) https://www.pressmantech.com/tech/3993
* `/proc/meminfo` の主な項目
    ```
    MemTotal:       24521792 kB   # 物理メモリの総量
    MemFree:          380616 kB   # 完全に未使用のメモリ
    MemAvailable:   20517936 kB   # 新たに利用可能と推定されるメモリ
    Buffers:               4 kB   # ブロックデバイス用のバッファキャッシュ
    Cached:         20528596 kB   # ページキャッシュ
    SwapTotal:      12386300 kB
    SwapFree:        9692140 kB
    Dirty:               196 kB   # ディスクへの書き戻し待ち
    AnonPages:       2603280 kB   # 無名ページ（プロセスのヒープ等）
    Slab:             576504 kB   # カーネルのデータ構造用
    ```

## 負荷テスト
* ab（Apache Bench）
    * Webサーバーの負荷テストツール。
    ```
    ab -n 100 -c 10 https://example.com/
    ```
    * `-n`（requests） 総リクエスト数
    * `-c`（concurrency） 同時リクエスト数
    * `-T`（content-type） POSTやPUTを行う時のContent-Typeを指定（デフォルトは `text/plain`）
    * `-H` 追加のHTTPヘッダを指定
    * `-k` Keep-Aliveを使用する
    * POSTのテスト
    ```
    ab -n 100 -c 10 -p postTest.json -T "application/json" https://example.com/graphql
    ```
    * (?) abは古くからあるツールで単一スレッドのため、高い負荷をかける用途にはwrkやk6、vegetaなどの方が向いている。
* 負荷テスト中は、サーバー側で `vmstat` や `top` を回してリソースの推移を確認するとよい。

## macOSのメモリ使用状況の確認
* 結論としては、アクティビティモニタを見るのが一番早い。
    * ただし、中身の細かいところを理解したい場合は、以下の `top` / `vm_stat` / `sysctl` で見てみるのもよい。
* `top` コマンドの表示項目
    * `Load Avg: 2.28, 2.39, 2.86`
        * 実行待ちを含むジョブ数（1分平均、5分平均、15分平均）。
    * `MemRegions: 253907 total, 3304M resident, 154M private, 2733M shared.`
        * メモリリージョンの数と内訳。
    * `PhysMem: 15G used (4566M wired), 1334M unused.`
        * 使用中の物理メモリサイズ。
        * `wired` はカーネルによって確保され、スワップアウトできない領域。アクティビティモニタの「確保されているメモリ」に対応する(?)。
        * アクティビティモニタの「使用済みメモリ」と値が異なるのは、アクティビティモニタがキャッシュされたメモリを含めないため。
    * `VM: 3706G vsize, 2308M framework vsize, 283324954(0) swapins, 286726869(0) swapouts.`
        * 仮想メモリの総サイズ、フレームワークが消費する仮想メモリサイズ、スワップインを起こしたページ数、スワップアウトを起こしたページ数。
        * 仮想メモリとは、実メモリ上にページを配置したり（スワップイン）、抱えきれなくなったページをディスク上に退避したり（スワップアウト）することで、プロセスからは膨大なメモリが使用可能なように見せる仕組み。当然、実メモリよりも大幅に大きなサイズになる。
    * `Networks: packets: 29295924/24G in, 28867267/10G out.`
        * ネットワークに対するin/outのパケット数とデータサイズ。
    * `Disks: 228412322/4196G read, 28975599/1733G written.`
        * ディスク装置に対するread/writeのデータサイズ。
    * (参考) https://zudoh.com/linux/mac-top-command
* `vm_stat`
    * 仮想メモリの使用状況を確認できる。単位はページなので、1ページあたりのバイト数（4096）を掛けて換算する。
    * アクティビティモニタとの対応関係（いずれも推測を含む(?)）
        * `Pages occupied by compressor` が「圧縮メモリ」
        * `Pages wired down` が「確保されているメモリ」
        * wired、active、inactive、speculative、occupied by compressor の合計が「キャッシュされたファイル」と「使用済メモリ」の合計にあたり、`top` の `PhysMem` の used と一致する
        * `Pages purgeable` と `File-backed pages` の合計がキャッシュにあたり、「キャッシュされたファイル」に対応する
    * (参考) https://songmu.jp/riji/entry/2015-05-08-mac-memory.html
* `sysctl` で様々な値を確認できる。
    * `sysctl -a` … 全て
    * `sysctl hw.memsize` … 物理メモリの総量
    * `sysctl -n hw.physicalcpu_max` … 物理コア数
    * `sysctl -n hw.logicalcpu_max` … 論理コア数
    * `-n` は値だけを表示するオプション。
* メモリが不足したときのmacOSの挙動
    * まず「メモリ圧縮」が行われる。アクティブではないメモリ領域を圧縮してスペースを空ける機能だが、この圧縮／伸張はプロセッサが行うため、その分パフォーマンスが低下する。
    * さらに不足すると「スワッピング」が行われる。アクティブではないメモリ領域をストレージにファイルとして退避させ、物理メモリの空きを増やす。
    * スワッピングが発生すると、メモリとストレージ間の読み書きに時間がかかる。そのためメモリ不足による速度低下は、「エンコードが遅くなる」といった継続的な処理よりも、「ファイルを開く／保存」「アプリの起動や終了」といったタイミングで顕著に現れる。
