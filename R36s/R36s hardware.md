# R36S ハードウェア備忘録

## 1. 目的

R36SをLinux PCとして利用するにあたり、実機で確認したハードウェア特性を記録する。

キッティング手順とは分離し、本書では主に以下を扱う。

- RK3326のCPU性能と動作クロック
- Raspberry Pi Zero 2 Wとの比較
- microSDインターフェースの性能
- microSDカード交換による性能変化
- Linux PCとして利用する際のボトルネック

測定環境：

```text
R36S
Armbian-unofficial / Ubuntu 24.04
Linux 6.12.32-lts-rk3326
aarch64
RAM 1GB
```

---

# 2. R36SのCPU

R36SはRockchip RK3326を搭載する。

```text
CPU     : ARM Cortex-A35 ×4
ISA     : ARMv8-A / aarch64
RAM     : 1GB
```

RK3326は一般に最大1.5GHzとして紹介されることがあるが、今回使用しているR36S-Armbian環境で確認した最大クロックは異なる。

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
```

結果：

```text
1296000
1296000
```

したがって、本環境でOSから利用可能な最大クロックは、

**1.296GHz**

である。

---

# 3. CPU governor

標準状態：

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

結果：

```text
ondemand
```

この状態で、

```bash
sysbench cpu --threads=4 --time=30 run
```

を実行しながら、

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

を監視すると、負荷中も、

```text
1008000
```

すなわち **1.008GHz** で動作し続けた。

つまり今回のArmbianでは、`ondemand` governorがCPU最大クロック1.296GHzまで上げていなかった。

これはCPUベンチマーク結果にかなり大きく影響する。

---

# 4. sysbench CPU

## 4.1 R36S — ondemand

### 1 thread

```bash
sysbench cpu --threads=1 --time=30 run
```

```text
441.70 events/sec
```

### 4 threads

```bash
sysbench cpu --threads=4 --time=30 run
```

```text
1562.16 events/sec
```

この状態ではCPUクロックは約1.008GHzだった。

---

# 5. performance governorでの測定

CPU最大性能そのものを見るため、governorを `performance` として測定した。

最大クロック：

```text
1.296GHz
```

## 5.1 1 thread

```text
570.80 events/sec
```

## 5.2 4 threads

```text
2292.11 events/sec
```

4 thread / 1 thread：

```text
2292.11 / 570.80 ≒ 4.02
```

ほぼ4コア分きれいにスケールしている。

---

# 6. Raspberry Pi Zero 2 WとのCPU比較

比較対象：

```text
Raspberry Pi Zero 2 W
Cortex-A53 ×4
約1.0GHz
RAM 512MB
64-bit Raspberry Pi OS
```

同一のsysbench CPUテストを実施した。

| CPU test | Pi Zero 2 W | R36S ondemand | R36S performance |
| --- | ---: | ---: | ---: |
| 1 thread | 495.04 | 441.70 | **570.80** |
| 4 threads | 1970.43 | 1562.16 | **2292.11** |
| 4T / 1T | 3.98倍 | 3.54倍 | **4.02倍** |

最大クロックで比較すると、

```text
1 thread
R36S / Pi = 570.80 / 495.04
           ≒ 1.153
```

→ **R36S 約15.3%高速**

```text
4 threads
R36S / Pi = 2292.11 / 1970.43
           ≒ 1.163
```

→ **R36S 約16.3%高速**

となった。

---

# 7. CPU比較から分かったこと

当初、

```text
Cortex-A35 1.296GHz
vs
Cortex-A53 1GHz
```

という仕様だけを見ると、性能差を予想しにくかった。

実測では、

- 同程度の約1GHz動作ではCortex-A53のPi Zero 2 Wが速い
- R36Sを1.296GHzまで動かすとR36Sが逆転する

という結果になった。

つまり、

> **Cortex-A53の方がクロック当たりの性能は高いが、R36Sは約30%高いクロックでそれを補い、最終的な整数演算性能では約15〜16%上回る**

という関係と考えられる。

---

# 8. ondemand設定の影響

R36Sのsysbench結果：

```text
ondemand / 約1.008GHz
1T   441.70
4T  1562.16
```

performance / 1.296GHz：

```text
1T   570.80
4T  2292.11
```

特に4threadでは、

```text
2292.11 / 1562.16 ≒ 1.47
```

となり、約47%性能が上昇した。

したがってR36Sでは、

> **CPU自体の性能だけでなく、cpufreq governorの設定が実性能へ大きく影響する**

ことに注意する。

通常運用で常時 `performance` にするかは、消費電力・発熱・バッテリー時間とのトレードオフになる。

---

# 9. CPU温度

7-Zip benchmark実行後：

```bash
cat /sys/class/thermal/thermal_zone0/temp
```

結果：

```text
47272
```

= **47.3℃**

少なくとも今回の測定では、CPU温度によるthermal throttlingを疑う状態ではなかった。

連続監視：

```bash
watch -n 1 "awk '{printf \"%.1f°C\n\",\$1/1000}' /sys/class/thermal/thermal_zone0/temp"
```

---

# 10. メモリ性能

sysbench memory：

```bash
sysbench memory --threads=1 --time=30 run
sysbench memory --threads=4 --time=30 run
```

結果：

| Memory write | Pi Zero 2 W | R36S |
| --- | ---: | ---: |
| 1 thread | **911 MiB/s** | 608 MiB/s |
| 4 threads | **3552 MiB/s** | 2494 MiB/s |

Pi Zero 2 Wの方が明確に高い。

ただしR36S側は `ondemand` governor時に測定した値なので、最大CPUクロック条件での再測定値ではない。

一方、容量は、

```text
Pi Zero 2 W : 512MB
R36S        : 1GB
```

なので、

> **Pi Zero 2 Wはメモリ帯域、R36Sはメモリ容量**

という違いがある。

Linuxデスクトップとしては、R36Sの1GB RAMは実用上大きなメリットになる。

---

# 11. 7-Zip benchmark

## Raspberry Pi Zero 2 W

7-Zip 25.01：

```text
Compress     1732 MIPS
Decompress   5297 MIPS
Total        3514 MIPS
```

## R36S

7-Zip 23.01：

```text
Compress     1134 MIPS
Decompress   2865 MIPS
Total        1999 MIPS
```

ただし、この値は**単純比較しない**。

理由：

- 7-Zipのバージョンが異なる
- R36S側はDict 22〜25
- Pi側はDict 22〜23
- R36Sはbenchmark中約945〜968MHzと表示された
- R36SはRAM使用量889MB / 947MB
- Dict 25 compressionで著しい性能低下が発生

R36S：

```text
RAM size  : 947MB
RAM usage : 889MB
```

特にDict 25：

```text
Compression Speed : 151 KiB/s
Rating            : 173 MIPS
```

まで落ちており、1GB RAMの限界に近づいた影響が大きいと考えられる。

7-ZipはCPUだけではなく、

- RAM容量
- メモリ帯域
- Dictionaryサイズ
- 7-Zipバージョン

の影響を受けるため、参考値として扱う。

---

# 12. SHA-256性能

OpenSSL：

```bash
openssl speed -seconds 10 sha256
```

結果：

| Block | Pi Zero 2 W | R36S |
| --- | ---: | ---: |
| 16B | 4.60 MB/s | **5.68 MB/s** |
| 64B | 14.20 MB/s | **20.86 MB/s** |
| 256B | 34.37 MB/s | **65.88 MB/s** |
| 1KiB | 53.52 MB/s | **141.14 MB/s** |
| 8KiB | 63.83 MB/s | **212.22 MB/s** |
| 16KiB | 64.69 MB/s | **219.85 MB/s** |

16KiBでは、

```text
219.85 / 64.69 ≒ 3.40
```

となり、R36Sが約3.4倍高速。

---

# 13. SHA性能差の理由

7-Zipが表示したCPU feature：

R36S：

```text
hwcap:8FF
CRC32
SHA1
SHA2
AES
ASIMD
```

Pi Zero 2 W：

```text
hwcap:887
CRC32
ASIMD
```

R36Sでは、

- SHA1
- SHA2
- AES

のARM Crypto ExtensionがOSから利用可能になっている。

そのためSHA-256では、通常の整数演算性能とは全く異なる結果となった。

つまりCPU性能は、

> **単純なクロックや「Cortex-A35対A53」だけでは判断できず、対象処理がどのハードウェア命令を利用できるかでも大きく変わる**

ことが分かる。

OpenSSL自体のバージョンも異なるため厳密なCPUコア比較ではないが、実際のLinux環境で利用できるSHA-256性能としては有効な比較である。

---

# 14. CPU性能まとめ

現時点の整理：

| 特性 | 優位 |
| --- | --- |
| 約1GHz同士の一般整数処理 | Pi Zero 2 W |
| R36S最大1.296GHzでの整数処理 | **R36S 約15〜16%** |
| メモリ帯域 | Pi Zero 2 W |
| RAM容量 | **R36S 1GB** |
| SHA-256 | **R36S 大幅優位** |
| ARM Crypto Extension | **R36S** |
| governorによる性能変動 | R36Sで大きい |

したがって、

> **「R36SはPi Zero 2 Wより速い／遅い」と一括りにはできない。**

通常の整数処理ではCortex-A53の効率が高いが、R36Sは高クロックで逆転する。

一方、専用命令を利用できる処理ではR36Sが数倍高速になる場合もある。

---

# 15. microSDホスト性能

R36SのmicroSDインターフェース：

```bash
sudo cat /sys/kernel/debug/mmc1/ios
```

結果：

```text
clock:          150000000 Hz
actual clock:   150000000 Hz
bus width:      4 bits
timing spec:    sd uhs SDR104
signal voltage: 1.80 V
```

つまり、

- UHS-I
- SDR104
- 4bit
- 150MHz
- 1.8V signaling

で動作している。

---

# 16. R36S上でのmicroSD理論帯域

150MHz × 4bit：

```text
150,000,000 × 4 / 8
= 75,000,000 byte/sec
```

したがって、この構成での単純なバス帯域上限は、

**約75MB/s**

となる。

SDR104規格自体は208MHz / 104MB/s級だが、このR36Sでは150MHzで動作している。

実際のI/Oにはprotocol overheadがあるため、70MB/s弱出れば非常に良好と考えられる。

---

# 17. 旧128GB microSD

交換前に使用していたmicroSDカードは以下。

```text
TOSHIBA M203
型番    : THN-M203K1280C4
容量    : 128GB
規格    : microSDXC / UHS-I / Class 10 / U1
最大読込: 100MB/s
発売時期: 2018年7月
```

東芝ブランド時代のスタンダードクラスのmicroSDXCカードであり、購入から約8年経過した個体である。

したがって、今回の比較は単純な

> 「安価な旧カードから高速カードへ交換した」

というものではない。

M203は発売当時としては一般用途向けの正規メーカー製UHS-Iカードであり、現在でも読み込み性能は十分高い。

---

## 17.1 容量・パーティション構成

カード実容量：

```text
公称容量 : 128GB
実容量   : 116.77GiB
```

交換前の構成：

```text
/dev/mmcblk1
├─ p1  FAT32    256MiB   /boot/u-boot
└─ p2  ext4     116.5GiB /
```

root filesystemの実使用量：

```text
約5.7GB
```

であり、128GBの大部分は未使用だった。

このため、新カード選定では容量よりもLinux root filesystemとしてのI/O特性を優先することにした。

---

# 18. TOSHIBA M203の実測性能

## 18.1 Sequential Read

```bash
sudo hdparm -t /dev/mmcblk1
```

結果：

```text
61.21 MB/s
```

direct read：

```text
68.0 MB/s
```

R36S側のmicroSDインターフェースは、

```text
SDR104
150MHz
4bit
```

で動作しており、単純なバス帯域上限は約75MB/s。

したがって、M203は約8年前の製品でありながら、

> **読み込みについては現在でもR36S側の実用上限近くまで性能を出している**

と評価できる。

---

## 18.2 Sequential Write

1GiBファイルによる実測：

```text
4.9 MB/s
```

読み込みの68MB/sに対して、書き込み性能は大幅に低い。

ただし、この値を

> 「TOSHIBA M203という製品本来の書き込み性能」

とみなすべきではない。

今回測定したのは約8年間使用した実個体であり、

- 使用履歴
- NANDの状態
- 内部GC
- 消去済みブロックの状態
- filesystem利用履歴

などが現在のwrite性能へ影響している可能性がある。

したがって本書では、あくまで

> **8年間使用したTHN-M203K1280C4の現在の実機性能**

として記録する。

---

# 19. TOSHIBA M203の4K Random I/O

fio条件：

```bash
fio \
 --name=rand4k \
 --filename=$HOME/sdtest.bin \
 --size=512M \
 --rw=randrw \
 --rwmixread=70 \
 --bs=4k \
 --direct=1 \
 --iodepth=1 \
 --runtime=30 \
 --time_based \
 --group_reporting
```

結果：

```text
Read  : 114 IOPS / 456 KiB/s
Write :  50 IOPS / 203 KiB/s
```

Write latency：

```text
平均    : 18.25 ms
p99     : 約1.25秒
最大    : 約1.40秒
```

ここで重要なのは、平均転送速度そのものよりも、

> **一定割合のwriteで1秒を超えるlong-tail latencyが発生していること**

である。

---

# 20. Linux root filesystemとして見たM203

M203が設計された時代の主用途は、

- 写真保存
- 動画保存
- スマートフォン等の容量拡張
- 比較的大きなファイルの連続アクセス

といった用途である。

一方、今回のR36SではmicroSDをLinuxのroot filesystemとして使用する。

Linuxデスクトップでは、

- package管理
- 多数の小ファイル展開
- browser cache
- SQLite
- log
- 設定ファイル
- metadata更新
- compile時の一時ファイル

など、小さいrandom read/writeが頻繁に発生する。

この用途では、

```text
Sequential Read
```

だけでなく、

```text
4K Random IOPS
Write latency
p95 / p99 latency
```

が体感性能に大きく影響する。

したがってM203の弱点は、

> **読み込み速度ではなく、Linux root filesystemとして使用した場合のwrite/random I/O特性**

にあったと考える。

---

# 21. 新microSDの選定

交換先として、

**KIOXIA EXCERIA G3 32GB**

を選択した。

仕様：

```text
容量       : 32GB
Class      : Class 10
UHS        : UHS-I / U3
Video      : V30
Application: A2
最大読込   : 160MB/s
最大書込   : 50MB/s
```

容量は128GBから32GBへ大幅に減少する。

しかし実際のroot使用量は約5.7GBであり、32GBでも十分余裕がある。

今回の選定では容量より、

- U3
- V30
- A2
- write性能
- random I/O
- latency安定性

を重視した。

---

# 22. M203とEXCERIA G3の世代・用途差

両カードの違いは、単純な最大読込速度だけではない。

```text
TOSHIBA M203
UHS-I
Class 10
U1
最大読込100MB/s
2018年世代

        ↓

KIOXIA EXCERIA G3
UHS-I
U3
V30
A2
最大読込160MB/s
最大書込50MB/s
```

特に今回重要なのは `A2` である。

Application Performance Classは、小規模random accessを含むアプリケーション用途を意識した性能クラスであり、Linux root filesystemの利用形態にも近い。

したがって今回の交換は、

> **「最大read性能を上げる」ためではなく、「Linux PC用途に適したwrite/random I/O特性へ変更する」**

ことを目的とした。

---

# 23. 32GBカードへの移行

旧カードのroot partition：

```text
/dev/mmcblk1p2
116.5GiB
```

に対して実使用量は約5.7GB。

このため、移行前にext4 filesystemとpartitionを、

```text
116.5GiB
   ↓
20GiB
```

へ縮小した。

縮小後：

```text
Device      Start       End       Sectors
p1          32768       557055      524288
p2         557056     42500095    41943040
```

`p2` の開始位置は変更せず、右端のみ縮小した。

ディスク先頭から`p2`終端まで約20.27GiBなので、

```text
MBR
↓
boot領域
↓
p1 FAT32
↓
p2 ext4
```

までを含む約20GBのdisk imageとして取得し、32GBカードへ移行した。

---

# 24. KIOXIA EXCERIA G3実測

## 24.1 Sequential Read

`hdparm`：

```text
60.75 MB/s
```

direct read：

```text
64.8 MB/s
```

旧M203：

```text
61.21 MB/s
68.0 MB/s
```

新EXCERIA G3：

```text
60.75 MB/s
64.8 MB/s
```

読み込み性能は実質的に同等。

むしろ旧M203の方がdirect readではわずかに高い。

これは、

> **M203のread性能が現在でも十分高く、旧カードの時点ですでにR36S側SDホスト性能の上限付近まで達していた**

ことを示している。

したがって、最大読込160MB/sというEXCERIA G3のスペックはR36S上では使い切れない。

---

## 24.2 Sequential Write

旧M203：

```text
4.9 MB/s
```

EXCERIA G3：

```text
44.2 MB/s
```

比率：

```text
44.2 / 4.9 ≒ 9.0
```

**約9倍**

となった。

EXCERIA G3の公称最大50MB/sに対し、R36S上でも44.2MB/sを確認できた。

SDホスト側がreadで約65～68MB/s出せていることを考えると、write側でもカード性能をかなり有効に利用できている。

---

# 25. 4K Random I/O比較

同一fio条件で比較した。

| 指標 | TOSHIBA M203 128GB | KIOXIA EXCERIA G3 32GB | 変化 |
| --- | ---: | ---: | ---: |
| Random Read | 114 IOPS | **369 IOPS** | 約3.2倍 |
| Random Read BW | 456 KiB/s | **1477 KiB/s** | 約3.2倍 |
| Random Write | 50 IOPS | **158 IOPS** | 約3.2倍 |
| Random Write BW | 203 KiB/s | **635 KiB/s** | 約3.1倍 |

最大read性能ではほとんど差がなかった一方、小規模random I/Oでは約3倍の差が出た。

これはLinux PCとして利用する場合に重要な改善である。

---

# 26. Write latency比較

## TOSHIBA M203

```text
平均      18.25 ms
p99       約1250 ms
最大      約1400 ms
```

## KIOXIA EXCERIA G3

```text
平均       4.18 ms
p99        9.37 ms
p99.5     13.04 ms
p99.9     28.18 ms
最大      約224 ms
```

特にp99：

```text
約1250ms
   ↓
9.37ms
```

となり、約133分の1まで短縮した。

---

## 26.1 中央値から分かること

興味深いことに、random writeの中央値は、

```text
M203       約3 ms
EXCERIA    約3.9 ms
```

で、大幅な差はない。

つまりM203は、

> **常に遅いカードだったわけではない。**

通常のwriteは数msで処理できている一方、

```text
時々1秒以上停止する
```

というlong-tail latencyが大きな問題だった。

EXCERIA G3では、

```text
p95      約6.8ms
p99      約9.4ms
p99.5    約13ms
p99.9    約28ms
```

と、write latencyの分布が大幅に安定した。

Linuxデスクトップの体感改善には、この差がSequential Write以上に効く可能性がある。

---

# 27. microSD交換の評価

実測結果：

| 指標 | TOSHIBA M203 | EXCERIA G3 |
| --- | ---: | ---: |
| 容量 | 128GB | 32GB |
| Sequential Read | 68.0 MB/s | 64.8 MB/s |
| Sequential Write | 4.9 MB/s | **44.2 MB/s** |
| 4K Random Read | 114 IOPS | **369 IOPS** |
| 4K Random Write | 50 IOPS | **158 IOPS** |
| Write平均Latency | 18.25 ms | **4.18 ms** |
| Write p99 | 約1.25秒 | **9.37 ms** |

今回の交換を、

> 「8年前の遅いSDカードを新しい高速カードへ交換した」

と表現するのは適切ではない。

TOSHIBA M203はread性能に関しては現在でも十分高速であり、R36S上ではEXCERIA G3とほぼ同等だった。

本質的な違いは、

```text
大容量・連続read中心の旧世代カード
             ↓
Linux root用途にも適したrandom/write性能を持つA2/V30カード
```

への変更である。

---

# 28. microSDに関する結論

今回の測定から、R36SでmicroSDをLinux root filesystemとして使う場合、

> **最大読込速度だけでカードを選んでも意味が薄い**

ことが分かった。

R36SのSDホストは、

```text
SDR104
150MHz
4bit
```

で、実測readは約65～68MB/s。

したがって、

```text
100MB/s
160MB/s
200MB/s
```

といったカード側の最大read値は、R36Sでは直接性能差になりにくい。

むしろ重要なのは、

- Sequential Write
- 4K Random Read
- 4K Random Write
- write latency
- p95 / p99 latency
- Application Performance Class

である。

---

# 29. R36SをLinux PCとして見た場合のストレージ選定

今回の結果から、R36S用途では、

```text
容量
   より
I/O品質
```

を優先する価値が高い。

実際に、

```text
128GB
↓
32GB
```

へ容量を4分の1に減らしたが、

```text
Sequential Write   約9倍
Random I/O         約3倍
Write p99 latency  約1/133
```

となった。

root使用量が約5.7GBであれば、32GBでも十分な余裕がある。

したがって、R36SをLinux PCとして利用する場合、

> **必要以上に大容量なカードより、A1/A2やU3/V30等を含め、小規模I/Oとwrite品質の良いカードを選ぶ方が実用性能向上につながる**

と考える。

---

# 30. CPU・microSDを合わせた現時点の評価

今回の実測から、R36Sの性能上の特徴は次のように整理できる。

```text
CPU
├─ Cortex-A35 ×4
├─ OS上限 1.296GHz
├─ ondemandでは約1.008GHzに留まる
├─ performance時はPi Zero 2 Wより約15～16%高速
└─ SHA2/AES等のCrypto Extensionが利用可能

RAM
├─ 1GB
├─ Pi Zero 2 Wの2倍の容量
└─ memory bandwidth自体はPi Zero 2 Wが高速

microSD host
├─ UHS-I SDR104
├─ 150MHz / 4bit
├─ 理論約75MB/s
└─ 実測read 約65～68MB/s

microSD card
├─ 旧 TOSHIBA M203
│   ├─ 2018年世代
│   ├─ read性能は現在でも十分
│   └─ Linux用途ではwrite latencyが弱点
│
└─ KIOXIA EXCERIA G3
    ├─ A2 / U3 / V30
    ├─ Sequential Write 約44MB/s
    ├─ Random I/O 約3倍
    └─ p99 write latencyを大幅改善
```

R36SをLinux PCとして使う際には、

> **CPUではgovernor設定、ストレージでは最大read値よりrandom/write特性**

を見ることが重要である。