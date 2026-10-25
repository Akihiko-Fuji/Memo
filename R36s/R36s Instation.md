# R36S Armbian キッティング手順

## 1. このキッティングの目的

R36Sをゲーム専用機としてではなく、**携帯可能な小型Linux PCとして利用する**ことを目的とする。
R36Sは本来レトロゲーム用の携帯端末だが、ハードウェアとして見ると、

- 64bit ARM SoC
- 1GB RAM
- 3.5インチ 640×480 IPS液晶
- ケース
- バッテリー
- 十字キー、ABXY、アナログスティック等の入力装置
- 2つのmicroSDスロット
- USB-C OTG
- オーディオ
- Linuxを起動可能なハードウェア

が最初から一体になっている。
今回の購入個体は64GB microSD付きで[概ね5,000円程度](https://ja.aliexpress.com/w/wholesale-R36s-%E6%AD%A3%E8%A6%8F.html?spm=a2g0o.detail.search.0)で購入が可能である。
このハードウェアへArmbianを導入し、

- XFCEデスクトップ
- 日本語入力
- ソフトウェアキーボード
- SSH
- Wi-Fi
- Bluetooth
- Linuxアプリケーションのnative build環境

を追加する。

---

# 2. Raspberry Pi Zero 2 Wとの比較

この用途で比較対象になるのは、Raspberry Pi Zero 2 Wである。Raspberry Pi Zero 2 Wは、

- 4コア Cortex-A53 1GHz
- 512MB RAM
- Wi-Fi 2.4GHz
- Bluetooth 4.2 / BLE
- microSD ×1
- USB OTG
- GPIO
- mini HDMI

を備えた小型SBCである。一方、標準的なR36Sは、

- RK3326 / Cortex-A35 4コア 最大1.5GHz
- 1GB DDR3L
- 3.5インチ 640×480 IPS液晶
- microSD ×2
- USB-C OTG
- バッテリー
- ケース
- スピーカー
- ゲームコントローラー一式

を備える。比較すると以下のようになる。

| 項目 | R36S | Raspberry Pi Zero 2 W |
| --- | --- | --- |
| CPU | RK3326 / Cortex-A35 ×4 / 最大1.5GHz | Cortex-A53 ×4 / 1GHz |
| RAM | 1GB DDR3L | 512MB |
| 液晶 | 3.5インチ 640×480 IPS | なし |
| ケース | 標準装備 | 別途必要 |
| バッテリー | 標準装備 | 別途必要 |
| 操作系 | D-pad、ABXY、2スティック等 | なし |
| microSD | 2スロット | 1スロット |
| USB OTG | USB-C | micro USB |
| Wi-Fi | なし | 2.4GHz Wi-Fi内蔵 |
| Bluetooth | なし | Bluetooth 4.2内蔵 |
| GPIO | 実質利用対象外 | 40pin |
| 映像出力 | なし | mini HDMI |

CPUについてはR36SのRK3326の方が動作クロックは高いが、Cortex-A35とCortex-A53ではCPUコアのアーキテクチャが異なるため、**あらゆる処理でR36Sの方が高速という意味ではない**。
PC化という用途で明確なメリットになるのは、

> **1GB RAMと、液晶・ケース・バッテリー・入力装置・ストレージスロットまで含めた完成した携帯端末が約5,000円程度で手に入ること**

である。つまりR36Sは、

> **「液晶とバッテリーとコントローラーまで付いた格安ARM SBC」**

として見ることができる。

一方で最大の弱点は、

- Wi-Fiなし
- Bluetoothなし
- OTGポートの給電能力に余裕が少ない

ことである。そこで今回、

> **Wi-FiとBluetoothを1本のUSBレシーバーへ集約する**

ことを重要な設計条件とした。

---

# 3. R36Sのクローン・別ハードウェアに注意

## 3.1 「R36S」という名前だけではハードウェアを特定できない

R36Sには多数の互換機・クローン・外観だけ似せた別ハードウェアが存在する。標準的なR36SはRockchip RK3326を使用しているが、市場には、

- RK3326を使う別基板
- RK3128
- Allwinner A33
- MTK6572系

など、異なるSoCを搭載した「R36S」も存在する。特に後者のような**SoC自体が異なるモデルは、標準R36S用のArkOSやArmbianをそのまま使用できない**。

したがって、

> **R36S-Armbianを書き込む前に、対象端末が標準的なRK3326系R36Sであることを確認する**

必要がある。

---

## 3.2 オリジナル系を見分ける目安

コミュニティでよく使われている確認ポイントとして、

### RAMチップ

標準的なR36Sでは、

> **RAMチップが2枚実装され、合計1GB**

となっている個体が多い。一方、初期のクローンにはRAMチップ1枚のものが多く、中には512MBしか搭載していないものもある。ただし現在では、

> **RAMチップ2枚のクローンも存在する**

ため、これだけでオリジナルと断定してはいけない。

---

### microSDを抜いて起動する

標準的なR36SはOSをmicroSDから起動し、基本的にOS用内蔵ストレージを持たない。そのため、

> **microSDをすべて抜いて電源を入れても、通常はOSやエラーメニューまで起動しない**

ことが標準R36Sを判断する一つの目安になる。一部クローンは内蔵ストレージ等から起動し、

```text
We can't find any systems!
```

等のメッセージを表示するものが確認されている。

---

### そのほかの確認材料

- SoCがRK3326
- RAMが1GB
- 標準ArkOS系が起動する
- EASYROMS partitionを使用
- クローン固有のDTBではない
- microSDなしではOSが起動しない
- 基板・RAM実装を確認する

など、複数の条件を合わせて判断する。

> **「RAMチップが2枚だから絶対にオリジナル」という判定はしない。**

本手順は、**RK3326 / 1GB RAMの標準R36S系個体**を対象とする。

---

# 4. Armbianの導入

R36S対応Armbianとして、

**R36S-Stuff / R36S-Armbian**

を使用する。

Repository:

`R36S-Stuff/R36S-Armbian`

Releaseは同Repositoryの `Releases` から取得する。R36S-Stuff側のREADMEでも、Releaseの配布先として同RepositoryのReleaseページが案内されている。今回使用したのはRC5。

使用環境：

```text
Armbian-unofficial 25.08.0-trunk noble
Ubuntu 24.04 LTS
Linux 6.12.32-lts-rk3326
aarch64
```

配布イメージが `.xz` の場合、Rufus等でそのままmicroSDへ書き込める。書き込み後、R36SのOS用microSDスロットへ挿入して起動する。

---

# 5. 初期確認

Architecture：

```bash
uname -m
```

期待値：

```text
aarch64
```

Kernel：

```bash
uname -r
```

今回：

```text
6.12.32-lts-rk3326
```

OS：

```bash
cat /etc/os-release
```

ここでR36S向けKernelで正常に起動していることを確認してから、追加設定へ進む。

---

# 6. microSD容量の拡張

OSイメージを書き込んだ直後は、microSD全体までroot partitionが拡張されていない場合がある。

確認：

```bash
df -h /
lsblk
```

今回のroot partition：

```text
/dev/mmcblk1p2
```

partitionをmicroSD末尾まで拡張した後、

```bash
sudo resize2fs /dev/mmcblk1p2
```

でext4 filesystemを拡張する。

確認：

```bash
df -h /
```

今回、128GB級microSDで約115GBのroot filesystemを利用できる状態となった。

---

# 7. 基本パッケージ

```bash
sudo apt update
```

```bash
sudo apt install -y git build-essential dkms
```

R36S-Armbianは一般的なUbuntu PCとは異なり、Kernel、DTB、U-Boot等がR36S向けに調整されている。

そのため、

> **必要なアプリケーションは更新しても、Kernel周辺を無条件に最新版へ変更しない**

ことを基本方針とする。

---

# 8. 日本語入力

日本語入力には、

- Fcitx5
- Mozc

を使用する。

```bash
sudo apt install -y fcitx5 fcitx5-mozc
```

```bash
im-config -n fcitx5
```

ログアウトまたは再起動する。

設定GUI：

```bash
fcitx5-configtool
```

---

# 9. ソフトウェアキーボード

## 9.1 目的

R36Sを携帯PCとして利用する以上、

> **USBキーボードを接続しなくても最低限の文字入力ができる**

状態にしておきたい。

特にUSB OTGはAX900で使用するため、日常操作のためだけにUSBキーボードを常時接続する構成にはしない。

---

## 9.2 xvkbdの導入

```bash
sudo apt install -y xvkbd
```

起動：

```bash
xvkbd
```

---

## 9.3 XFCEパネルから起動できるようにする

画面が小さいため、xvkbdを常時表示するのではなく、

> **必要なときだけパネルから呼び出す**

方式とする。

XFCEパネル上で、

1. パネルを右クリック
2. `パネル`
3. `新しいアイテムの追加`
4. `ランチャー`
5. ランチャーのプロパティを開く
6. 新しい項目を追加

以下を設定する。

```text
名前:
xvkbd

コマンド:
xvkbd
```

これで画面上のアイコンからソフトウェアキーボードを起動できる。

---

## 9.4 自動起動する場合

必要であれば、

```bash
mkdir -p ~/.config/autostart
```

```bash
nano ~/.config/autostart/xvkbd.desktop
```

内容：

```ini
[Desktop Entry]
Type=Application
Name=xvkbd
Exec=xvkbd
OnlyShowIn=XFCE;
X-GNOME-Autostart-enabled=true
```

ただしR36Sでは表示面積が限られるため、パネルランチャー方式を推奨する。

---

## 9.5 日本語入力

xvkbdは日本語変換を行うアプリではない。

構成は、

```text
xvkbd
  ↓
キー入力
  ↓
Fcitx5
  ↓
Mozc
  ↓
日本語変換
```

となる。

---

## 9.6 Onboardの削除

標準で導入されていたOnboardは使用しないため削除する。

```bash
sudo apt purge -y onboard onboard-common
```

不要依存を確認：

```bash
sudo apt autoremove --dry-run
```

問題なければ：

```bash
sudo apt autoremove
```

---

# 10. 初期ネットワークにはBuffalo WLI-UC-GNMを使用する

初期セットアップではAX900ではなく、動作実績がある**Buffalo WLI-UC-GNM**を使用する。
これは最終構成ではなく、**セットアップと復旧のためのアダプタ**である。理由は単純で、

> **Linux Kernel標準の `rt2800usb` で動作し、追加ドライバなしでネットワークへ接続できる**

ためである。

USB ID：

```text
0411:01a2
```

Driver：

```text
rt2800usb
```

認識：

```bash
lsusb
```

Driver：

```bash
lsmod | grep rt2800
```

必要なら：

```bash
sudo modprobe rt2800usb
```

NetworkManager：

```bash
nmcli d
```

ここでWi-Fiを接続し、その後の作業はSSHから実施する。

---

# 11. なぜ最終構成をAX900にするのか

標準R36SにはWi-FiもBluetoothも搭載されていない。USBレシーバーを使えば追加できるが、

```text
Wi-Fi USB dongle
+
Bluetooth USB dongle
```

のように2デバイスへ分けると、

- USB Hubへの給電が必要
- 消費電力増加
- ケーブル増加
- 携帯性低下
- OTG給電負荷増加

となる。

さらに今回のR36Sでは、実機評価からOTG側の給電余力が大きくないと判断した。そこで、

> **Wi-Fi + Bluetoothを1本のUSBレシーバーで実現できるAX900**

を採用する。AX900採用の主目的は単純な通信速度ではなく、

> **給電能力に余裕の少ないOTGポートで、Wi-FiとBluetoothをUSB 1デバイスへ集約すること**

である。

---

# 12. AX900のハードウェア

今回使用したAX900はAICSemi AIC8800D80系。購入した製品は[これ](https://ja.aliexpress.com/item/1005012250661281.html)であるが、必ずしも動作するとは限らない。

実機判定：

```text
chip_id=7
chip_mcu_id=1
```

このためDriverは、

```text
legacy-mcu1
```

branchを使用する。同じAX900の商品名でも内部chip revisionが異なる可能性があるため、別個体へ展開する場合は注意する。

---

# 13. AX900ビルド環境

```bash
sudo apt install -y \
  dkms \
  build-essential \
  mokutil \
  eject \
  usb-modeswitch \
  dwarves \
  pahole
```

確認：

```bash
pahole --version
```

今回：

```text
v1.25
```

---

# 14. Kernel headers確認

```bash
ls -l /lib/modules/$(uname -r)/build
```

今回：

```text
/lib/modules/6.12.32-lts-rk3326/build
 -> /usr/src/linux-headers-6.12.32-lts-rk3326
```

---

# 15. R36S-Armbian RC5のBTF設定補正

## 15.1 目的

外部Kernel moduleは、実行中Kernelと同じ構成を前提としてビルドする必要がある。今回のRC5では、

```text
CONFIG_DEBUG_INFO_BTF
CONFIG_DEBUG_INFO_BTF_MODULES
```

について、

- 実Kernel
- 配布headers

の設定差が確認された。

特に `CONFIG_DEBUG_INFO_BTF_MODULES` はKernel module内部構造にも影響するため、DKMS build前に一致させる。

---

## 15.2 実Kernel設定

```bash
KVER=$(uname -r)
HDR=/usr/src/linux-headers-$KVER
```

```bash
zcat /proc/config.gz > /tmp/running-kernel.config
```

```bash
grep '^CONFIG_DEBUG_INFO_BTF' /tmp/running-kernel.config
```

今回：

```text
CONFIG_DEBUG_INFO_BTF=y
CONFIG_DEBUG_INFO_BTF_MODULES=y
```

---

## 15.3 headers確認

```bash
grep '^CONFIG_DEBUG_INFO_BTF' "$HDR/.config"
```

```bash
grep CONFIG_DEBUG_INFO_BTF \
  "$HDR/include/generated/autoconf.h"
```

```bash
grep CONFIG_DEBUG_INFO_BTF \
  "$HDR/include/config/auto.conf"
```

実Kernelと一致していれば変更不要。

---

## 15.4 不一致の場合

バックアップ：

```bash
sudo cp "$HDR/.config" "$HDR/.config.original"
sudo cp "$HDR/include/generated/autoconf.h" \
        "$HDR/include/generated/autoconf.h.original"
sudo cp "$HDR/include/config/auto.conf" \
        "$HDR/include/config/auto.conf.original"
```

実Kernel設定：

```bash
sudo cp /tmp/running-kernel.config "$HDR/.config"
```

`autoconf.h`：

```bash
grep -q '^#define CONFIG_DEBUG_INFO_BTF 1$' \
  "$HDR/include/generated/autoconf.h" \
  || echo '#define CONFIG_DEBUG_INFO_BTF 1' \
  | sudo tee -a "$HDR/include/generated/autoconf.h"
```

```bash
grep -q '^#define CONFIG_DEBUG_INFO_BTF_MODULES 1$' \
  "$HDR/include/generated/autoconf.h" \
  || echo '#define CONFIG_DEBUG_INFO_BTF_MODULES 1' \
  | sudo tee -a "$HDR/include/generated/autoconf.h"
```

`auto.conf`：

```bash
grep -q '^CONFIG_DEBUG_INFO_BTF=y$' \
  "$HDR/include/config/auto.conf" \
  || echo 'CONFIG_DEBUG_INFO_BTF=y' \
  | sudo tee -a "$HDR/include/config/auto.conf"
```

```bash
grep -q '^CONFIG_DEBUG_INFO_BTF_MODULES=y$' \
  "$HDR/include/config/auto.conf" \
  || echo 'CONFIG_DEBUG_INFO_BTF_MODULES=y' \
  | sudo tee -a "$HDR/include/config/auto.conf"
```

---

## 15.5 注意

このheadersパッケージは完全なKernel source treeではない。

したがって、

```bash
make prepare
```

```bash
make modules_prepare
```

による再生成は行わない。

---

# 16. AX900 Driver取得

Driver：

`shenmintao/aic8800d80`

```bash
cd ~
git clone https://github.com/shenmintao/aic8800d80.git
cd ~/aic8800d80
```

今回の個体：

```bash
git switch legacy-mcu1
git pull
```

確認：

```bash
git branch --show-current
```

```text
legacy-mcu1
```

---

# 17. Driverのインストール

インストール中はAX900を接続せず、

```text
R36S
  │
  └─ Buffalo WLI-UC-GNM
       └─ rt2800usb
```

のみとする。これにより、

- SSH接続を維持
- AX900初期化とDriver installを分離
- OTGへ複数無線アダプタを同時接続しない

という状態を作る。

```bash
cd ~/aic8800d80
chmod +x install.sh
sudo ./install.sh
```

成功時：

```text
Module built successfully.
Module installed via DKMS.
Wi-Fi module loaded successfully.
Wi-Fi module is active in kernel.
```

確認：

```bash
dkms status
```

今回：

```text
aic8800/1.0.0, 6.12.32-lts-rk3326, aarch64: installed
```

---

# 18. インストーラ末尾のBluetooth確認

AX900をまだ挿していないため、

```text
Checking for Bluetooth interfaces on combo adapters...
```

でBluetooth device待ちになる場合がある。以下まで終了していれば、

```text
DKMS module registered
Wi-Fi kernel module is loaded
Firmware installed
```

主要なインストール処理は終了している。その場合は `Ctrl+C` で終了しても問題ない。

---

# 19. BuffaloからAX900への交換

ホットスワップではなく、一度完全にシャットダウンする。

```bash
sudo poweroff
```

電源OFF後、

1. Buffaloを抜く
2. AX900を挿す
3. R36Sを起動

する。

これにより最終構成そのものを評価する。

---

# 20. AX900のUSB mode switch

AX900は起動時、

```text
a69c:5721
Aic MSC
    ↓
a69c:8d80
AIC Wlan
    ↓
368b:8d81
AIC 8800D80
```

とUSB deviceを再列挙する。Driver installerにより、

- udev
- usb_modeswitch
- firmware loader

が導入されるため、通常運用では手動 `eject` は不要。

---

# 21. Wi-Fi

Driver：

```text
aic8800_fdrv
```

確認：

```bash
nmcli d
```

GUIではXFCEのネットワークアイコンからSSIDを選択する。

---

# 22. Bluetooth

BluetoothはLinux標準の、

```text
btusb
```

を使用する。

確認：

```bash
lsusb -t
```

正常例：

```text
/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=dwc2/1p, 480M
    |__ Port 001: Dev xxx, If 0, Class=Wireless, Driver=btusb, 480M
    |__ Port 001: Dev xxx, If 1, Class=Wireless, Driver=btusb, 480M
    |__ Port 001: Dev xxx, If 2, Class=Vendor Specific Class, Driver=aic8800_fdrv, 480M
```

つまり1つのAX900を、

```text
AX900
 ├─ Wi-Fi
 │    └─ aic8800_fdrv
 │
 └─ Bluetooth
      └─ btusb
```

として使用する。

---

# 23. USB給電・安定性確認

AX900採用の重要な目的はUSB負荷の低減である。最終構成で、

```bash
sudo dmesg \
  | grep -iE 'under|voltage|over.current|error -71|reset high-speed|USB disconnect' \
  | tail -50
```

を確認する。

AX900は起動時にmode switchするため、

```text
USB disconnect
```

が数回出ること自体は正常。

一方、

```text
error -71
reset high-speed USB device
USB disconnect
over-current
undervoltage
```

が通常利用中に繰り返される場合は、給電またはUSB信号品質を疑う。今回のAX900単独構成では継続的なUSB異常は確認されなかった。

---

# 24. 最終構成

```text
R36S
│
├─ RK3326
├─ RAM 1GB
├─ 3.5" 640×480 IPS LCD
├─ Battery
├─ Controller
├─ microSD ×2
│
├─ Armbian / Ubuntu 24.04
│   ├─ XFCE
│   ├─ Fcitx5 + Mozc
│   ├─ xvkbd
│   └─ SSH
│
└─ USB-C OTG
     │
     └─ AX900 / AIC8800D80
          │
          ├─ Wi-Fi
          │    └─ aic8800_fdrv
          │
          └─ Bluetooth
               └─ btusb
```

予備：

```text
Buffalo WLI-UC-GNM
  └─ rt2800usb
```

---

# 25. Buffaloは復旧用として残す

AX900導入後、Buffaloは通常運用では使用しない。ただしLinux標準driverで動作するため、

> **Driver更新やKernel更新後にAX900が使えなくなった場合の復旧用Wi-Fi**

として保管する。必要なら、

```bash
sudo modprobe rt2800usb
```

だけでDriverをロードできる。

---

# 26. Kernel更新

AX900 DriverはDKMS登録されている。

```bash
dkms status
```

ただしR36Sでは、

- Kernel
- headers
- DTB
- U-Boot
- BSP

の組み合わせが重要である。したがって、

> **UbuntuだからKernelも無条件に最新版へ上げる**

という運用はしない。R36S対応状況を確認してから更新する。

---

# 27. NP2kaiによるLinux PCとしての確認

PC-98 emulatorであるNP2kaiもARM64 native buildできる。

```bash
git clone https://github.com/AZO234/NP2kai.git
cd NP2kai
```

```bash
cmake -S . -B build \
  -DBUILD_SDL=ON \
  -DBUILD_WX=OFF \
  -DBUILD_X=OFF \
  -DUSE_SDL=2
```

```bash
cmake --build build -j2
```

生成：

```text
build/sdlnp21kai_sdl2
```

これは、

> **R36Sが単にゲーム機用Linuxを動かしているだけではなく、通常のARM64 Linuxソフトウェアをbuildして実行できる**

ことの確認にもなる。

---
