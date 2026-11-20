# Khadas VIM1：Armbian microSD起動の検証記録・技術的知見・次の作業

- **更新日**：2026年11月20日
- **対象機種**：Khadas **VIM1（初代、VIM1Sではない）**／基板リビジョン `VIM1.V14`
- **目的**：microSD上のArmbian 26.8.11を起動し、可能であれば最終的にeMMCで運用する
- **現在の到達点**：microSDのファイルシステム互換性問題は解消。microSD上のカーネル・initrd・DTBをU-Bootから読み込める。しかし、**Vendor U-Bootの `booti` が例外終了し、ArmbianのLinuxカーネルには制御が渡っていない**
- **次に進む方向**：旧Vendor U-Bootの調整をいったん終了し、Khadas公開の**VIM1用Mainline U-Boot**を検証して切り替える。eMMCへの書き込みはイメージ形式・実行ツール・復旧方法を確認してから行う

> **確度の区分**：以下では **【実機確認】**、**【推定】**、**【未確認】** を区別する。実験で分かったことと、次の仮説を混同しない。

## 1. 環境・使用したもの

| 項目 | 内容 |
|---|---|
| SBC | Khadas VIM1（初代、S905X/GXL、RAM 2GiB、eMMC約8GB、VIM1.V14） |
| eMMCの現行OS | Khadas Fenix 1.1.1、Ubuntu 18.04.6 LTS、Linux `4.9.241` |
| eMMCの現行U-Boot | Vendor U-Boot `2015.01`、ビルド `Jul 25 2022 - 15:16:53` |
| 初期起動段 | BL2ビルド `Aug 28 2019`、BL31ビルド `May 22 2019`。UARTログでeMMCからのFIP/BL30/BL31/BL33読み込みを確認 |
| Armbian | `Armbian_26.8.11_Khadas-vim1_resolute_current_6.18.57_xfce_desktop.img.xz`（配布イメージ） |
| microSD | 64GB級（Raspberry Pi上では約57GiB）。MBR、先頭セクター8192の単一ext4パーティション |
| microSDのラベル／UUID | `armbi_root`／`6a597149-e131-41c8-806d-078ea2a66162` |
| Armbianカーネル | `6.18.57-current-meson64` |
| Armbian用DTB | `/boot/dtb/amlogic/meson-gxl-s905x-khadas-vim.dtb`（42,212バイト） |
| 作業PC | Windows、WSL2 Ubuntu 24.04、Raspberry Pi 4B（ARM64） |
| UART | 3.3V TTL UART、115200bps、8N1。U-Bootプロンプトは `kvim#` |
| 重要なデバイス対応 | **U-Boot**では `mmc 0`＝microSD、`mmc 1`＝eMMC。**Raspberry Pi**のカードリーダーでは `/dev/sda`／`/dev/sda1`。**VIM1上のLinux**では過去の確認で `/dev/mmcblk0`＝eMMC、`/dev/mmcblk1`＝microSD |

**注意**：デバイス名は環境と起動状況で変わる。`mmc 0` と `/dev/mmcblk0` を同一視しない。書き込み前には必ず実機で再確認する。

## 2. これまでの検証経過

### 2.1 microSDの作成・通常起動試験

1. Armbian Imagerで上記ArmbianイメージをmicroSDに書き込んだ。
2. Raspberry PiでMBRとext4パーティション、`/boot/Image`、`boot.scr`、`boot.cmd`、`armbianEnv.txt`、VIM1用DTBの存在を確認した。【実機確認】
3. SDの先頭領域にはKhadas由来の `VIM1.u-boot.sd.bin` に関連する識別情報があり、ArmbianイメージがSD起動向けU-Bootを含むことを確認した。【実機確認】
4. eMMCのOSをAndroid 7.1.2、公式Android 9、公式Ubuntu 18.04.6と変えても、**通常の電源投入ではeMMC側のOSを起動**し、Armbian SDへ切り替わらなかった。【実機確認】
5. FUNCTION/RESETやRecovery等の操作も試行したが、目的とするArmbianの通常起動には至らなかった。Khadas公式のUpgrade Modeと通常のmicroSD起動は別の操作である。

### 2.2 書き込み・復旧経路の確立

eMMCを変更する際の復旧方法を先に検証した。

| 方法 | 結果 |
|---|---|
| Windows USB Burning Tool 2.2.0 | VIM1を `1b8e:c003`（WorldCup）として認識したが、約4%で `[0x10302001] Romcode/Switch status/Check connect/Enum device error` |
| WSL2 + usbipd-win + Khadas `burn-tool` | DDR初期化は成功するが、U-Boot実行段階でUSB接続が切断され失敗。WSL2のUSB再接続が問題となった |
| **Raspberry Pi 4B 直結 + Khadas `utils`** | **公式Android 9およびUbuntu 18.04.6のeMMC書き込み・起動に成功** |

Raspberry Pi上で成功した、**Amlogicパッケージ形式の純正Ubuntuイメージを書き込むコマンド**：

```bash
cd ~/utils
sudo ./aml-flash-tool/flash-tool --img=./ubuntu.img --parts=all --wipe --soc=gxl
```

- `~/utils` は `https://github.com/khadas/utils` を取得したもの。
- `aml-flash-tool/tools/linux-arm/update` は32-bit ARMの静的リンク実行ファイルで、Raspberry Pi 4B ARM64環境で動作した。
- 実際のログで `Writing bootloader [OK]`、各パーティションの書き込み、`Resetting board [OK]` を確認した。
- **【復旧実績あり】**：VIM1がUpgrade/WorldCup Modeに入れれば、Piから公式Ubuntuへ再書き込みできた。
- **重要**：`flash-tool` にそのまま `Armbian_...img` を指定してはいけない。これはMBR/ext4の**raw SDディスクイメージ**であり、純正Ubuntu/Androidの**Amlogic書き込みパッケージ**と形式が異なる。
- Raspberry Piのログには過去に低電圧警告もあった。次回の書き込み時はPiおよびVIM1の電源安定性に注意する。

### 2.3 旧eMMCブートローダーの確認

UARTで初期起動を取得すると、次のとおりだった。

```text
eMMC boot @ 1
Load fip header from eMMC ... part: 1
Load bl30 from eMMC ... part: 1
Load bl31 from eMMC ... part: 1
Load bl33 from eMMC ... part: 1
U-Boot 2015.01 (Jul 25 2022 - 15:16:53)
```

**【実機確認】** 電源投入時に実行されるブートローダーはeMMC側のVendor U-Boot。

過去にLinux上の `/dev/bootloader`（約4MiB）を退避してゼロクリアしたが、その後もUbuntuが起動した。**どのeMMC領域が実際のBL2/FIPの供給元なのかは未特定**。特に `eMMC boot @ 1` を、Linuxの `/dev/mmcblk0boot0`／`boot1` に無条件に対応付けないこと。**eMMCのboot0/boot1を推測で消去しない。**

### 2.4 U-Bootの起動設定とデバイス対応

自動起動をUARTから止め、`kvim#` で確認した。

```text
kvim#version
U-Boot 2015.01 (Jul 25 2022 - 15:16:53)
kvim#mmc list
SDIO Port B: 0
SDIO Port C: 1
kvim#printenv bootcmd
bootcmd=run storeboot
kvim#printenv boot_targets
## Error: "boot_targets" not defined
```

`mmc dev 0` / `mmc part` はMBR・Armbian SD、`mmc dev 1` / `mmc part` はAML形式・eMMCと確定した。

`storeboot` の主要処理：

```text
run setup_ethmac
cfgload
if load mmc 0:1 1020000 s905_autoscript ||
   load mmc 1:1 1020000 s905_autoscript ||
   load mmc 1:5 1020000 /boot/s905_autoscript;
then autoscr 1020000; fi
# 失敗時はeMMCのカーネル等を読み込んで起動
ext4load mmc 1:5 1080000 zImage
ext4load mmc 1:5 10000000 uInitrd
ext4load mmc 1:5 20000000 dtb.img
booti 1080000 10000000 20000000
```

- **【実機確認】** Vendor U-BootはmicroSDを全く見ていないわけではない。まず `s905_autoscript` を探し、結果としてeMMC Ubuntuへフォールバックしている。
- Armbianイメージの起動スクリプト名は `/boot/boot.scr`。既存の探索経路にそのまま適合していない。
- U-Bootの `boot_targets`、`fdtfile`、`scriptaddr`、`devtype`、`devnum`、`prefix` 等は初期状態では未定義。Mainline向け `boot.cmd` の動作には適切な変数が必要。
- `autoscr`、`booti`、`bootm`、`fdt` のコマンドが存在することを確認した。一方、`iminfo` はこのビルドにない。

### 2.5 microSDのext4互換性問題を特定・解消

**初期症状**（Vendor U-Boot）：

```text
kvim#ext4ls mmc 0:1 /
<DIR> 590 boot
kvim#ext4ls mmc 0:1 /boot
< ? > 0
kvim#ext4load mmc 0:1 0x32000000 /boot/boot.scr
** File not found /boot/boot.scr **
```

Raspberry Piで `/boot/boot.scr` の存在を確認しても同じ。切り分け用にルート直下へコピーした `/uboot-test.scr` は、Raspberry Piで **8,133バイト**なのにU-Bootで **0バイト**と表示された。

アンマウントした状態で `e2fsck -fn /dev/sda1` を実行し、ファイルシステムの破損は報告されなかった。元のext4の主要機能：

```text
... needs_recovery extent 64bit flex_bg metadata_csum_seed ... metadata_csum
```

**【実機確認】** テスト用メディアなのでバックアップは省略し、ファイルをPiへ一時退避した後、パーティション `/dev/sda1` のみ旧U-Boot互換を意識したext4で再作成した。

```bash
# Raspberry Pi側。デバイス名は実行前に要確認
sudo mkdir -p /home/pi/armbian-stage
sudo rsync -aHAX --numeric-ids /mnt/armbian/ /home/pi/armbian-stage/
# マウントを解除し、findmnt等で未マウントを確認してから
sudo mkfs.ext4 -F -b 4096 -I 256 \
  -L armbi_root \
  -U 6a597149-e131-41c8-806d-078ea2a66162 \
  -O ^64bit,^metadata_csum,^metadata_csum_seed \
  /dev/sda1
sudo mount /dev/sda1 /mnt/armbian
sudo rsync -aHAX --numeric-ids /home/pi/armbian-stage/ /mnt/armbian/
sync
```

- **パーティションテーブルとSD先頭のU-Boot領域は変更せず**、`/dev/sda1` のext4のみ再作成。
- 元のUUIDと `armbi_root` ラベルを維持。
- `mkfs.ext4` 実行時、SDが自動マウントされていて一度拒否されたが、アンマウント後に成功。拒否時には既存内容は消去されていなかった。
- 再作成後の `tune2fs -l` で `64bit`、`metadata_csum`、`metadata_csum_seed` が無効になったことを確認。

**修正後の実機結果**：

```text
kvim#ext4ls mmc 0:1 /boot
... 8133 boot.scr
kvim#ext4load mmc 0:1 0x32000000 /boot/boot.scr
8133 bytes read in 6 ms (1.3 MiB/s)
kvim#ext4load mmc 0:1 0x33000000 /uboot-test.scr
8133 bytes read in 6 ms (1.3 MiB/s)
```

**【結論】** 旧U-Bootが読めなかった原因はext4の機能・形式に関連している可能性が非常に高く、再作成で解消した。ただし、**3つの機能を一括で無効化したため、どれが単独の原因だったかは未特定**。

なお、過去に「SD未検出」というエラーが再発したときは、実際には**カードがソケットから抜けていた**。挿し直すと正常に初期化・読み込みできた。これを別のソフトウェア不具合として扱わない。

### 2.6 最新Armbianの起動ファイルを正常にロード

microSD挿入後、`kvim#` から次を実施した。

```text
kvim#mmc dev 0
kvim#ext4load mmc 0:1 0x10000000 /boot/uInitrd
33126842 bytes read in 1776 ms (17.8 MiB/s)
kvim#ext4load mmc 0:1 0x34000000 /boot/Image
37689856 bytes read in 2018 ms (17.8 MiB/s)
kvim#ext4load mmc 0:1 0x20000000 /boot/dtb/amlogic/meson-gxl-s905x-khadas-vim.dtb
42212 bytes read in 20 ms (2 MiB/s)
```

**【実機確認】** カーネル、initrd、DTBの3ファイルはいずれも旧U-BootからRAMへ正常にロード可能。

Armbianの `boot.cmd`（ソース）は、modern kernel分岐で `uInitrd`、`Image`、`dtb/${fdtfile}` を読み込み、`booti` を実行する構成。デフォルトの `rootdev` は `/dev/mmcblk1p1`、実際のルート指定は `armbianEnv.txt` 等で上書きする設計。手動試験では固定UUIDを `bootargs` に渡した。

### 2.7 手動 `booti` がU-Boot内部でクラッシュ

1回目の手動起動：

```text
kvim#setenv bootargs root=UUID=6a597149-e131-41c8-806d-078ea2a66162 rootwait rootfstype=ext4 console=ttyAML0,115200 console=tty1 loglevel=7
kvim#booti 0x34000000 0x10000000 0x20000000
libfdt fdt_path_offset() returned FDT_ERR_NOTFOUND
[rsvmem] fdt get prop fail.
"Synchronous Abort" handler, esr 0x96000210
ELR: 77e9f1f4
LR:  77e9f1e8
Resetting CPU ...
```

- **【実機確認】** `Starting kernel ...` に至らず、Vendor U-Boot内部で例外終了。自動再起動後はeMMCのUbuntuに戻る。
- 初回のinitrd配置 `0x10000000` は、Mainline DTBに記載された予約領域 `secmon@10000000`（2MiB）と重複していたため、次回は別アドレスへ移動した。

### 2.8 `rsvmem` 用のDTB補正と再実験

DTBの `/reserved-memory` には以下が存在した。

```text
hwrom@0          0x00000000 / 0x01000000
secmon@10000000  0x10000000 / 0x00200000
secmon@5000000   0x05000000 / 0x00300000
secmon@5300000   0x05300000 / 0x02000000
linux,cma        0x10000000 (size要求)
```

`rsvmem dump` ではBL31の予約領域が `start=0x05000000`、`size=0x00300000`。Vendor U-Boot側が参照する疑いのある `/reserved-memory/linux,secmon` ノードがMainline DTBに見当たらなかった。

RAM上のDTB（`0x20000000`）だけを実験的に補正した。

```text
fdt addr 0x20000000
fdt resize 65536
fdt mknode /reserved-memory linux,secmon
fdt set /reserved-memory/linux,secmon compatible "amlogic, aml_secmon_memory"
fdt set /reserved-memory/linux,secmon reg <0x0 0x05000000 0x0 0x00300000>
fdt set /reserved-memory/linux,secmon no-map
setenv fdtaddr 0x20000000
rsvmem check
```

**【実機確認】** `rsvmem check` はエラーを表示せず終了した。ただし、同じ範囲を示す予約メモリノードを重複定義しており、これは**診断用の一時変更**で、SD/eMMCの恒久的なDTBには反映していない。

initrdを予約領域と重複しない `0x18000000` に再配置し、次のとおり再試行：

```text
kvim#ext4load mmc 0:1 0x18000000 /boot/uInitrd
33126842 bytes read in 1776 ms (17.8 MiB/s)
kvim#ext4load mmc 0:1 0x34000000 /boot/Image
37689856 bytes read in 2022 ms (17.8 MiB/s)
kvim#rsvmem check
kvim#booti 0x34000000 0x18000000 0x20000000
"Synchronous Abort" handler, esr 0x96000210
ELR: 77e9f1f4
LR:  77e9f1e8
```

**【結論】** `rsvmem` の表示エラーは消えたが、**まったく同じ例外アドレスでVendor U-Bootが再びクラッシュした**。`linux,secmon` ノード不足は不整合の一つだった可能性があるものの、**クラッシュの根本原因と断定できない**。

**【未確認】** 正確な原因はVendor U-Bootの該当ビルドのシンボル解析なしでは断定できない。`booti` 実装の互換性、FDT修正処理、legacy形式のinitrd受け渡しなど、ほかの要因は残る。

### 2.9 旧UbuntuのLinux起動後に見られたSD通信エラー

旧Ubuntuカーネル `4.9.241` の起動ログには、microSDで次のエラーが複数回出現した。

```text
meson-mmc: sd: resp_timeout
retry cmd 18
mmcblk1: error -110 sending status command
```

**【実機確認】** 旧カーネルがmicroSDアクセスに失敗している。一方、U-BootではSDから合計数十MiBのファイルを正常に読み込めたため、**物理的なSDカード故障と即断しない**。Linuxドライバー・通信タイミング・カードの相性・電源などは、Armbian起動後の別途検査事項。

## 3. 現在の判断：何が解決し、何が残ったか

| 課題 | 状態 | 根拠 |
|---|---|---|
| VIM1のUSB書き込み・復旧経路 | **解決** | Raspberry Piで公式OSを書き込み、起動確認 |
| UARTによるブートログ取得・U-Boot操作 | **解決** | `kvim#` で操作可能 |
| 起動中のU-Bootの所在 | **判明** | BL2/FIP/BL33をeMMCからロード |
| SDカード認識とデバイス番号 | **判明** | U-Bootの`mmc 0`がSD、`mmc 1`がeMMC |
| 旧U-Bootによるext4読み取り | **解決** | ext4再作成後、`boot.scr` 8,133B読込成功 |
| Armbianカーネル等のロード | **解決** | `Image` 37,689,856B、`uInitrd` 33,126,842B、DTB 42,212Bをロード |
| 自動起動スクリプト互換性 | **未解決** | Vendor `storeboot` は `s905_autoscript` を探索 |
| Vendor U-Bootでの `booti` | **失敗・再現性あり** | 同一ELR `0x77e9f1f4` でSynchronous Abort |
| Armbianカーネル起動・rootfsマウント | **未到達** | `Starting kernel` に至らない |
| SDカード通信安定性（新Linux環境） | **未確認** | 旧Ubuntuカーネルではタイムアウトあり |
| eMMCへのArmbianインストール | **未着手** | まずSDから起動成功が必要 |

### 技術的な判断

- **【強く支持】** 初期のSD読み取り不能はext4の新しい機能の組合せによる旧U-Boot非互換。再フォーマットで改善した。
- **【実機確認】** 旧Vendor U-Bootは最新Armbianのファイルを読み込めても、`booti` 段階で例外終了する。
- **【推定】** Vendor U-Bootの独自処理とMainline向けDTB／カーネルの相互運用性が不足している。該当命令位置の厳密な原因は不明。
- **【方針】** これ以上Vendor U-Boot向けに場当たり的なDTB変更を加えず、Mainline U-Bootへ移行する道を優先する。

## 4. 次に実施すること（推奨手順）

### Phase 1 — 配布物・書き込み方法を検証する（最優先、まだ書き込まない）

Khadasが公開している **`khadas-uboot` リリース `0.17.3`** を候補とする。

- リリースURL：<https://github.com/khadas/khadas-uboot/releases/tag/0.17.3>
- 候補ファイル：`VIM1.uboot-mainline.emmc.aml.img`
- リリース記載のU-Boot系列：`2021.07+`（Khadasのパッチ込み）
- リリースに記載されたブート探索順：**SPI → USB → SD → NVMe → MMC(eMMC) → PXE → DHCP**。
- リリースには `aml-burn-tool` 向けのeMMC用パッケージ、および別のSD/SPI/Krescue向けイメージがある。
- Armbianのビルド構成 `config/sources/families/meson-gxl.conf` にも、初代VIM1向けに同じKhadasリリース `0.17.3` の事前ビルドU-Bootを使用する記述がある。

**実施項目**：

1. VIM1 **初代用**のeMMCパッケージを取得する（VIM1S/VIM2/VIM3用と取り違えない）。
2. `file`、ファイルサイズ、リリース付属チェックサム一覧（`list.md5sum` 等）で整合性を確認する。
3. `.aml.img` が `aml-burn-tool` 用の**Amlogicパッケージ**であること、使用ツールとオプションを確認する。
4. Raspberry Pi 4B上にある `~/utils` の `aml-burn-tool` の引数・実行方法を、`--help` とリリース手順で照合する。
5. **Mainline U-Bootを書き込むと現行のeMMC Ubuntuが起動しなくなる可能性がある**ため、リカバリーに使う公式Ubuntuの `ubuntu.img` と、成功済み `flash-tool` コマンドを残す。
6. SDの先頭領域に既に入っているMainline用U-Bootとのバージョン差、現在の `boot.scr` への対応（起動探索パス、DTB、initrd）を確認する。

公式リリースに記載された**参考書式**：

```bash
# 参考：実機書き込みコマンドとしては未実行・未確定
aml-burn-tool -b VIM1 -i VIM1.uboot-mainline.emmc.aml.img
```

> **注意**：このコマンドはリリース記載の例から具体化したもの。Raspberry Pi上での実行ファイル・起動条件・実際の書き込み範囲を確認する前に実行しない。既に成功した `flash-tool --parts=all --wipe` と同じ効果を仮定しない。

### Phase 2 — Mainline U-Bootへの切り替えとSD起動

Phase 1で方式が確認できた後に進む。

1. 安定した給電とUART記録を用意し、VIM1をUSB Upgrade/WorldCup Modeへ移行する。
2. **VIM1用Mainline U-BootのeMMC書き込みパッケージのみ**を正しいツールで適用する。
3. UARTで新しいU-Bootのバージョン、起動元、SD探索を確認する。
4. microSDのArmbianが起動したか検証する。`Starting kernel` ではなく、**Linux 6.18.57 のログ→rootfsマウント→ログイン**までを成功条件とする。
5. 成功時は次を確認する。

```bash
cat /etc/os-release
uname -r
findmnt /
lsblk -f
dmesg | grep -Ei 'mmc|sd:|error|timeout'
```

6. 起動できない場合は、まずU-Boot→起動スクリプト→DTB→カーネル→initramfs→rootfsのどこで停止したかをUARTログで確定する。必要ならSD上のブート構成を調整する。

**重要な注意点**：リリースに書かれた **U-Boot自身の起動デバイス探索順**と、SoCのBootROMが最初にどの媒体からBL2を読み込むかは**別の話**。Mainline U-Bootを書き込めば必ずmicroSD上のブートローダーそのものが先に実行される、という意味ではない。新U-BootがSDのArmbianを選択できることを実機で確認する。

### Phase 3 — ArmbianをeMMCへ導入（SD起動成功後）

1. Armbian上でeMMCデバイスを特定し、microSDと取り違えない。
2. Armbianが当該ボードで案内するインストール方法・eMMCへの書き込み方式を確認する。
3. eMMCの現行Ubuntuを消去することを前提に、インストール計画を定める。
4. SDなしの単独起動と、再起動・ネットワーク・ストレージ・電源投入時の安定性を検証する。

## 5. 禁止・注意事項

- **`/dev/mmcblk0boot0`／`boot1` の盲目的な消去、`mmc write`、`store erase`、`env save`／`saveenv` は実施しない。** 書き込み箇所を特定してから行う。
- **microSDの `64bit`・`metadata_csum`・`metadata_csum_seed` を再有効化しない。** 現行の互換ext4で旧U-Bootから読めることを確認済み。
- 試験で追加した `/reserved-memory/linux,secmon` はRAM内の診断用変更。起動成功の証拠ではなく、メディアに永続化していない。重複した予約領域の定義を恒久採用しない。
- 以前の手動起動指定 `0x10000000` はDTB上の予約メモリに重複した。次の起動実験でも読み込み領域と実行領域の重複を避ける。
- U-Bootの `mmc 0` とLinuxの `/dev/mmcblk0` は対応が異なる。特にeMMC書き込み時は必ず再確認。
- Piに保存した `/home/pi/armbian-stage` はext4再構築時の一時コピー。使用中の媒体そのもののバックアップではない。ユーザー判断でテスト媒体のバックアップは不要とした。
- eMMC変更後にUbuntuを維持できるとは限らない。公式Ubuntuの再フラッシュができる状態を維持する。

## 6. 次回作業開始時のチェックリスト

- [x] UARTから `kvim#` に入れる
- [x] microSD上のArmbian起動ファイルをext4から読み取れる
- [x] `booti` でVendor U-Bootがクラッシュすることを再現した
- [x] Raspberry Piから純正UbuntuへのUSB復旧に成功済み
- [ ] Mainline U-Boot 0.17.3の **VIM1用eMMCパッケージ**を取得・検証する
- [ ] Pi用 `aml-burn-tool` の適用可能性・具体的な引数・書き込み範囲を確認する
- [ ] 復旧イメージの所在と安定給電を確認する
- [ ] Mainline U-Bootを導入し、UARTで新バージョンを確認する
- [ ] Armbian microSDからLinux 6.18.57が起動することを確認する
- [ ] 必要ならeMMC導入・SDなし起動を検証する

## 7. 参考リンク

- [Khadas Mainline U-Boot 0.17.3（配布物・書き込み方式・ブート探索順）](https://github.com/khadas/khadas-uboot/releases/tag/0.17.3)
- [Armbian ビルド設定：meson-gxl.conf（VIM1の事前ビルドU-Boot採用）](https://github.com/armbian/build/blob/main/config/sources/families/meson-gxl.conf)
- [Khadas公式：VIM1 Boot into Upgrade Mode](https://docs.khadas.com/products/sbc/vim1/install-os/boot-into-upgrade-mode)
- [Khadas公式コミュニティ：How to make a truly bootable SD card?](https://forum.khadas.com/t/how-to-make-a-truly-bootable-sd-card/2706/2)
- [Khadas USBフラッシュツール](https://github.com/khadas/utils)

---

**引き継ぎ要約**：Armbian SDのext4互換性を修正し、Vendor U-Bootから`Image`・`uInitrd`・DTBの読み込みに成功。しかし`booti`は同じSynchronous Abort（ELR `0x77e9f1f4`）を繰り返し、Linuxカーネル未起動。Piからの公式Ubuntu復旧経路は確立済み。**次回はVIM1用Mainline U-Boot `0.17.3`のeMMCパッケージを検証し、安全な導入方法を確定してから移行する。**
