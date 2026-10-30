# R36S ハードウェア備忘録(要約版)

## 目次

1. [目的と測定環境](#1-目的と測定環境)
2. [CPUとgovernor](#2-cpuとgovernor)
3. [Raspberry Pi Zero 2 Wとの比較](#3-raspberry-pi-zero-2-wとの比較)
4. [メモリ性能](#4-メモリ性能)
5. [7-Zip・SHA-256とCrypto Extension](#5-7-zipsha-256とcrypto-extension)
6. [CPU性能まとめ](#6-cpu性能まとめ)
7. [microSDホスト性能](#7-microsdホスト性能)
8. [旧カード TOSHIBA M203 の実測](#8-旧カード-toshiba-m203-の実測)
9. [新カード KIOXIA EXCERIA G3 の選定と移行](#9-新カード-kioxia-exceria-g3-の選定と移行)
10. [M203とG3の比較](#10-m203とg3の比較)
11. [microSDに関する結論](#11-microsdに関する結論)
12. [総合評価](#12-総合評価)

---

## 1. 目的と測定環境

R36SをLinux PCとして利用するにあたり、実機で確認したハードウェア特性を記録する。キッティング手順とは分離し、CPU性能、Pi Zero 2 Wとの比較、microSDの性能、カード交換による変化、ボトルネックを扱う。

```text
R36S
Armbian-unofficial / Ubuntu 24.04
Linux 6.12.32-lts-rk3326
aarch64 / RAM 1GB
```

---

## 2. CPUとgovernor

R36SはRockchip RK3326(Cortex-A35 ×4、ARMv8-A、RAM 1GB)を搭載する。一般に最大1.5GHzと紹介されるが、本環境の `scaling_max_freq` / `cpuinfo_max_freq` はともに1296000で、OSから使える最大クロックは**1.296GHz**である。

標準のgovernorは `ondemand` だが、`sysbench cpu --threads=4` の負荷中も `scaling_cur_freq` は**1.008GHz**のままで、最大クロックまで上がらなかった。これはベンチマーク結果に大きく影響する。

---

## 3. Raspberry Pi Zero 2 Wとの比較

比較対象は Pi Zero 2 W(Cortex-A53 ×4、約1.0GHz、RAM 512MB、64-bit Raspberry Pi OS)。同一の sysbench CPU テスト結果(events/sec):

| CPU test | Pi Zero 2 W | R36S ondemand | R36S performance |
| --- | ---: | ---: | ---: |
| 1 thread | 495.04 | 441.70 | **570.80** |
| 4 threads | 1970.43 | 1562.16 | **2292.11** |
| 4T / 1T | 3.98倍 | 3.54倍 | **4.02倍** |

- performance時(1.296GHz)はR36Sが1threadで約15.3%、4threadsで約16.3%高速。4コアはほぼ理想的にスケールする。
- 約1GHz同士ではPi Zero 2 Wが速く、Cortex-A53のクロック当たり性能の高さを、R36Sは約30%高いクロックで補って逆転している。
- ondemandからperformanceへの変更で、4threadsは約47%向上(2292.11 / 1562.16 ≒ 1.47)。governorの設定が実性能を大きく左右する。常時performanceにするかは、消費電力・発熱・バッテリー時間とのトレードオフ。
- 7-Zip benchmark後のCPU温度は47.3℃で、thermal throttlingは疑われない。

---

## 4. メモリ性能

sysbench memory(write)の結果:

| | Pi Zero 2 W | R36S |
| --- | ---: | ---: |
| 1 thread | **911 MiB/s** | 608 MiB/s |
| 4 threads | **3552 MiB/s** | 2494 MiB/s |

R36S側は `ondemand` 時の測定で、最大クロックでの再測定値ではない。要するに**Pi Zero 2 Wはメモリ帯域、R36Sはメモリ容量(1GB)**が強みで、Linuxデスクトップ用途では1GBの容量が実用上大きなメリットになる。

---

## 5. 7-Zip・SHA-256とCrypto Extension

### 7-Zip benchmark

| | Pi Zero 2 W (25.01) | R36S (23.01) |
| --- | ---: | ---: |
| Compress | 1732 MIPS | 1134 MIPS |
| Decompress | 5297 MIPS | 2865 MIPS |
| Total | 3514 MIPS | 1999 MIPS |

この値は**単純比較できない**。バージョンとDictionaryサイズが異なり、R36Sはbenchmark中945〜968MHzで動作し、RAMもほぼ使い切っていた(889MB / 947MB)。特にDict 25では151 KiB/s(173 MIPS)まで落ち、1GB RAMの限界に近づいた影響が大きいと考えられる。参考値として扱う。

### SHA-256(`openssl speed -seconds 10 sha256`)

| Block | Pi Zero 2 W | R36S |
| --- | ---: | ---: |
| 16B | 4.60 MB/s | **5.68 MB/s** |
| 1KiB | 53.52 MB/s | **141.14 MB/s** |
| 16KiB | 64.69 MB/s | **219.85 MB/s** |

16KiBではR36Sが約3.4倍高速。理由は、R36SではSHA1/SHA2/AESのARM Crypto ExtensionがOSから利用できる(hwcap:8FF)のに対し、Pi Zero 2 WはCRC32とASIMDのみ(hwcap:887)であるため。OpenSSLのバージョンが異なり厳密なコア比較ではないが、実際のLinux環境で使えるSHA-256性能としては有効な比較である。

---

## 6. CPU性能まとめ

| 特性 | 優位 |
| --- | --- |
| 約1GHz同士の一般整数処理 | Pi Zero 2 W |
| R36S最大1.296GHzでの整数処理 | **R36S 約15〜16%** |
| メモリ帯域 | Pi Zero 2 W |
| RAM容量 | **R36S 1GB** |
| SHA-256 / Crypto Extension | **R36S** |
| governorによる性能変動 | R36Sで大きい |

「R36SはPi Zero 2 Wより速い/遅い」とは一括りにできない。通常の整数処理ではA53の効率が高いがR36Sは高クロックで逆転し、専用命令を使える処理ではR36Sが数倍速くなる。

---

## 7. microSDホスト性能

`/sys/kernel/debug/mmc1/ios` の結果は、UHS-I SDR104、4bit、150MHz、1.8Vシグナリング。

```text
150,000,000 × 4 / 8 = 75,000,000 byte/sec
```

単純なバス帯域上限は**約75MB/s**(SDR104規格自体は208MHz / 104MB/s級だが、R36Sでは150MHz動作)。protocol overheadを考えると、70MB/s弱出れば非常に良好と言える。

---

## 8. 旧カード TOSHIBA M203 の実測

### 仕様と構成

```text
TOSHIBA M203 THN-M203K1280C4
128GB / microSDXC / UHS-I / Class 10 / U1
最大読込100MB/s / 2018年7月発売(使用約8年)
```

構成は p1 FAT32 256MiB(/boot/u-boot)、p2 ext4 116.5GiB(/)。root使用量は約5.7GBで、容量の大部分は未使用だった。そのため新カード選定では容量よりI/O特性を優先した。

### 実測

- Sequential Read:`hdparm` 61.21MB/s、direct read 68.0MB/s。約8年前のカードでも、R36Sの実用上限近くまで出ている。
- Sequential Write:**4.9MB/s**(1GiBファイル)。ただし約8年使用した個体の現状値であり、使用履歴、NANDの状態、内部GC等の影響があり得るため、製品本来の性能とはみなさない。
- 4K Random I/O(fio、randrw 70/30、bs=4k、direct=1、iodepth=1、30秒):Read 114 IOPS / 456 KiB/s、Write 50 IOPS / 203 KiB/s。
- Write latency:平均18.25ms、p99 約1.25秒、最大約1.40秒。

### Linux root filesystemとしての弱点

Linuxデスクトップでは、package管理、小ファイルの展開、browser cache、SQLite、log、metadata更新など、小さなrandom read/writeが頻繁に発生する。この用途ではSequential Readよりも**4K Random IOPS、Write latency、p95/p99 latency**が体感性能を左右する。M203の弱点は読み込み速度ではなく、**一定割合のwriteで1秒を超えるlong-tail latencyが発生すること**だった。

---

## 9. 新カード KIOXIA EXCERIA G3 の選定と移行

### 選定

```text
KIOXIA EXCERIA G3 32GB
Class 10 / UHS-I U3 / V30 / A2
最大読込160MB/s / 最大書込50MB/s
```

重視したのはU3、V30、A2、write性能、random I/O、latency安定性。特にA2(Application Performance Class)は小規模random accessを含む用途を想定した規格で、Linux rootの利用形態に近い。目的は「最大read性能の向上」ではなく、**Linux PC用途に適したwrite/random I/O特性への変更**である。

### 32GBへの移行

実使用量が約5.7GBのため、移行前にp2(ext4)を116.5GiBから20GiBへ縮小した(開始位置は変更せず右端のみ縮小)。

```text
p1   32768 - 557055     (524288 sectors)
p2  557056 - 42500095   (41943040 sectors)
```

先頭からp2終端までの約20.27GiBをdisk imageとして取得し、32GBカードへ書き込んだ。

---

## 10. M203とG3の比較

### Sequential

| | M203 | G3 |
| --- | ---: | ---: |
| `hdparm` Read | 61.21 MB/s | 60.75 MB/s |
| direct Read | 68.0 MB/s | 64.8 MB/s |
| Sequential Write | 4.9 MB/s | **44.2 MB/s(約9倍)** |

Readは実質同等で、M203の時点ですでにR36SのSDホスト上限付近に達していた。G3の最大読込160MB/sはR36S上では使い切れない。一方Writeは、G3の公称50MB/sに対して44.2MB/sを確認できた。

### 4K Random I/O

| 指標 | M203 | G3 | 変化 |
| --- | ---: | ---: | ---: |
| Random Read | 114 IOPS | **369 IOPS** | 約3.2倍 |
| Random Read BW | 456 KiB/s | **1477 KiB/s** | 約3.2倍 |
| Random Write | 50 IOPS | **158 IOPS** | 約3.2倍 |
| Random Write BW | 203 KiB/s | **635 KiB/s** | 約3.1倍 |

### Write latency

| | M203 | G3 |
| --- | ---: | ---: |
| 平均 | 18.25 ms | **4.18 ms** |
| p99 | 約1250 ms | **9.37 ms** |
| p99.9 | - | 28.18 ms |
| 最大 | 約1400 ms | 約224 ms |

p99は約133分の1に短縮した。なお、random writeの**中央値**はM203が約3ms、G3が約3.9msでほぼ差がない。つまりM203は常に遅いカードだったわけではなく、**時々1秒以上停止する**ことが問題だった。G3ではp95 約6.8ms、p99 約9.4ms、p99.5 約13msと、分布が大幅に安定した。体感改善にはSequential Write以上に効く可能性がある。

---

## 11. microSDに関する結論

今回の交換は「8年前の遅いカードを新しい高速カードに替えた」ものではない。M203はread性能では現在も十分高速で、本質的な違いは、

```text
大容量・連続read中心の旧世代カード
        ↓
Linux root用途にも適したrandom/write性能を持つA2/V30カード
```

への変更である。R36SでmicroSDをroot filesystemとして使う場合、最大読込速度(100/160/200MB/s等)だけで選んでも意味が薄い。見るべきは次の項目。

- Sequential Write
- 4K Random Read / Write
- write latency、p95 / p99 latency
- Application Performance Class

また、容量を128GBから32GBへ4分の1にしたにもかかわらず、Sequential Writeは約9倍、Random I/Oは約3倍、Write p99 latencyは約1/133になった。root使用量が約5.7GBなら32GBで十分であり、**必要以上に大容量なカードより、A1/A2やU3/V30等を含め、小規模I/Oとwrite品質の良いカードを選ぶ方が実用性能の向上につながる**。

---

## 12. 総合評価

```text
CPU
├─ Cortex-A35 ×4
├─ OS上限 1.3GHz(カタログスペック 1.5GHz)
├─ ondemandでは約1.0GHzに留まる
├─ performance時はPi Zero 2 Wより約15～16%高速
└─ SHA2/AES等のCrypto Extensionが利用可能

RAM
├─ 1GB(Pi Zero 2 Wの2倍)
└─ メモリ帯域はPi Zero 2 Wが高速

microSD host
├─ UHS-I SDR104 / 150MHz / 4bit
├─ 理論約75MB/s
└─ 実測read 約65～68MB/s(writeも同程度が期待できる)

microSD card
├─ 旧 TOSHIBA M203:read十分、write latencyが弱点
└─ KIOXIA EXCERIA G3(A2/U3/V30)
    ├─ Sequential Write 約44MB/s
    ├─ Random I/O 約3倍
    └─ p99 write latencyを大幅改善
```

R36SをLinux PCとして使う際は、**CPUではgovernor設定、ストレージでは最大read値よりrandom/write特性**を見ることが重要である。
