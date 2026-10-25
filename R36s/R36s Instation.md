# R36S Armbian キッティング手順

## 1. 目的

R36Sをゲーム専用機としてではなく、**小型のLinux PCとして利用できる環境**へセットアップする。

単にArmbianを起動するだけではなく、最終的に以下を満たすことを目標とする。

- XFCEによるデスクトップ利用
- 日本語入力
- 本体単独でも操作できるソフトウェアキーボード
- SSHによるリモート操作
- Wi-Fi
- Bluetooth
- USB機器をできるだけ少なくした構成
- Kernel更新後も再構築可能なDKMSベースのWi-Fiドライバ
- 必要に応じてLinuxアプリケーションをnative buildできる開発環境

最終構成は以下。

| 項目 | 構成 |
| --- | --- |
| 本体 | R36S |
| SoC | Rockchip RK3326 |
| OS | Armbian-unofficial / Ubuntu 24.04 Noble |
| Kernel | Linux 6.12.32-lts-rk3326 / aarch64 |
| Desktop | XFCE |
| Wi-Fi | AX900 / AIC8800D80系 |
| Bluetooth | AX900内蔵Bluetooth |
| Wi-Fi Driver | `aic8800_fdrv` |
| Bluetooth Driver | Linux標準 `btusb` |
| 日本語入力 | Fcitx5 + Mozc |
| ソフトウェアキーボード | `xvkbd` |
| セットアップ・復旧用Wi-Fi | Buffalo WLI-UC-GNM / `rt2800usb` |

---

# 2. 構成方針

今回のキッティングでは、R36Sの制約を踏まえて以下の方針とする。

## 2.1 標準機能をできるだけ利用する

R36Sは一般的なPCと異なり、独自Kernel、DTB、U-Boot等への依存が大きい。

そのため、

- OS標準ドライバで動くものは標準ドライバを使う
- Kernel自体を不用意に交換しない
- 外部ドライバはDKMSで管理する
- 動作中のKernelとheadersの整合性を確認する

ことを基本とする。

---

## 2.2 セットアップ用と最終構成を分ける

Wi-Fi設定を行うためにWi-Fiが必要、という循環を避けるため、初期セットアップでは

**Buffalo WLI-UC-GNM**

を使用する。

このアダプタはLinux Kernel標準の

```text
rt2800usb
```

で動作するため、**追加ドライバをインストールする前からネットワークへ接続できる**。

したがってBuffaloは、

- 初回セットアップ
- AX900ドライバ取得
- DKMSビルド
- SSH接続
- 障害時の復旧

に利用する「ブートストラップ用Wi-Fi」と位置付ける。

---

## 2.3 最終的にはAX900へ集約する

R36SのOTGポートはUSB機器への給電余力が大きくない。

Wi-Fi用USBアダプタとBluetooth用USBアダプタを別々に接続すると、

- USB機器数が増える
- USB Hubが必要になる
- 消費電力が増える
- 配線が複雑になる
- USBバスの安定性にも不利

となる。

そこで最終構成では、

**Wi-Fi + Bluetoothを1本のUSBレシーバーで提供するAX900**

へ集約する。

AX900を選択する主目的はWi-Fi速度だけではなく、

> **給電能力に余裕の少ないR36SのOTG環境で、Wi-FiとBluetoothをUSBレシーバー1本にまとめること**

である。

---

# 3. Armbianの導入

## 3.1 使用イメージ

R36S対応Armbianとして以下を使用する。

### R36S-Stuff / R36S-Armbian

Repository:

https://github.com/R36S-Stuff/R36S-Armbian

Release一覧:

https://github.com/R36S-Stuff/R36S-Armbian/releases

今回使用したRC5:

https://github.com/R36S-Stuff/R36S-Armbian/releases/tag/RC5

今回確認した環境：

```text
Armbian-unofficial 25.08.0-trunk noble
Ubuntu 24.04 LTS
Linux 6.12.32-lts-rk3326
aarch64
```

---

## 3.2 microSDへの書き込み

配布イメージが `.xz` 形式の場合、そのままRufus等でmicroSDへ書き込める。

Windowsでは例えば以下を使用できる。

- Rufus
- Raspberry Pi Imager
- balenaEtcher

書き込み後、microSDをR36Sへ装着して起動する。

---

# 4. 初期確認

Kernel：

```bash
uname -r
```

今回：

```text
6.12.32-lts-rk3326
```

CPU architecture：

```bash
uname -m
```

期待値：

```text
aarch64
```

OS：

```bash
cat /etc/os-release
```

この時点で、

> 「R36S用Kernelで起動していること」

を確認してから追加設定へ進む。

---

# 5. microSD容量の確認と拡張

## 5.1 意図

OSイメージの容量のままroot filesystemが残っていると、128GB等のmicroSDを使用しても数GBしか利用できない場合がある。

最初に確認しておく。

```bash
df -h /
```

```bash
lsblk
```

今回のroot partition：

```text
/dev/mmcblk1p2
```

root partitionがmicroSD末尾まで拡張されていなければ、確認の上で拡張する。

例：

```bash
sudo parted /dev/mmcblk1 --script resizepart 2 100%
```

ext4 filesystemも拡張する。

```bash
sudo resize2fs /dev/mmcblk1p2
```

確認：

```bash
df -h /
```

今回、約128GBのmicroSDで約115GBのroot領域を利用できる状態となった。

---

# 6. 基本パッケージ

パッケージ情報を更新する。

```bash
sudo apt update
```

開発・ビルド用：

```bash
sudo apt install -y git build-essential dkms
```

## 方針

R36S-Armbianは通常のUbuntu PCとは異なり、Kernel、DTB、U-Boot等がR36S向けに調整されている。

そのため、環境構築のために必要な個別パッケージは導入するが、

```bash
apt full-upgrade
```

等でKernel周辺を無条件に更新することは避ける。

---

# 7. 日本語入力

## 7.1 意図

R36SをLinux PCとして利用するため、日本語入力はOS標準に近い構成で用意する。

入力メソッド：

- Fcitx5
- Mozc

を使用する。

インストール：

```bash
sudo apt install -y fcitx5 fcitx5-mozc
```

Fcitx5を入力メソッドとして設定：

```bash
im-config -n fcitx5
```

その後、ログアウトまたは再起動する。

必要に応じて設定GUIを開く。

```bash
fcitx5-configtool
```

---

# 8. ソフトウェアキーボード

## 8.1 なぜ必要か

R36Sは物理キーボードを標準装備していない。

USBキーボードを接続すればPCとして操作できるが、

- USBポートを消費する
- OTG給電を消費する
- 外出先でキーボードを持ち歩く必要がある

ため、最低限の操作を本体単独で行えるようにしておく。

今回は `xvkbd` を使用する。

---

## 8.2 インストール

```bash
sudo apt install -y xvkbd
```

起動：

```bash
xvkbd
```

バックグラウンド起動する場合：

```bash
xvkbd &
```

---

## 8.3 XFCEパネルへ登録

常時自動起動すると画面を占有するため、通常は

> **必要なときだけXFCEパネルから呼び出す**

構成を推奨する。

XFCEのパネル上で、

1. パネルを右クリック
2. `パネル`
3. `新しいアイテムの追加`
4. `ランチャー` を追加
5. ランチャーのプロパティを開く
6. 新規アイテムを追加

以下を設定する。

```text
名前:
xvkbd

コマンド:
xvkbd
```

必要に応じてキーボードのアイコンを設定する。

これにより、R36S本体だけでもパネルをタップしてソフトウェアキーボードを起動できる。

---

## 8.4 自動起動させる場合

起動直後から常に表示したい場合のみAutostartへ登録する。

```bash
mkdir -p ~/.config/autostart
```

以下を作成する。

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

ただしR36Sの画面は小さいため、

> **自動起動よりパネルランチャー方式の方が使いやすい**

と考える。

---

## 8.5 日本語入力との関係

`xvkbd` 自体が日本語変換を行うわけではない。

役割分担は、

```text
xvkbd
  ↓
キーストローク入力
  ↓
Fcitx5
  ↓
Mozc
  ↓
日本語変換
```

となる。

---

## 8.6 Onboardの削除

標準で入っていたOnboardは今回の環境では使用しないため削除する。

```bash
sudo apt purge -y onboard onboard-common
```

不要依存パッケージは、削除前に確認する。

```bash
sudo apt autoremove --dry-run
```

問題がなければ、

```bash
sudo apt autoremove
```

を実行する。

---

# 9. 初期ネットワーク：Buffalo WLI-UC-GNM

## 9.1 なぜBuffaloを使用するのか

初期セットアップではAX900ではなくBuffalo WLI-UC-GNMを使用する。

理由は、

> **Linux Kernel標準の `rt2800usb` で動作し、追加ドライバなしでWi-Fiへ接続できるため**

である。

USB ID：

```text
0411:01a2
```

Driver：

```text
rt2800usb
```

確認：

```bash
lsusb
```

```bash
lsmod | grep rt2800
```

必要なら手動ロード：

```bash
sudo modprobe rt2800usb
```

NetworkManager：

```bash
nmcli d
```

Wi-Fiへ接続した後はSSHを使用すると、その後の大量のコマンド入力が容易になる。

---

# 10. AX900導入方針

AX900は最終的なWi-Fi/Bluetoothアダプタとして使用する。

今回確認した個体はAICSemi AIC8800D80系で、

```text
chip_id=7
chip_mcu_id=1
```

の個体である。

そのため、

```text
legacy-mcu1
```

branchを使用する。

> 同じAX900という商品名でも内部chip revisionが異なる可能性はあるため、別個体へ展開する場合はchip revisionを確認する。

今回のキッティング手順は、実際に動作確認できた `chip_mcu_id=1` の個体を対象とする。

---

# 11. AX900ビルド環境

必要パッケージ：

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

# 12. Kernel headers確認

外部Kernel moduleをDKMSでビルドするため、現在動作しているKernelに対応するheadersが必要となる。

確認：

```bash
ls -l /lib/modules/$(uname -r)/build
```

今回：

```text
/lib/modules/6.12.32-lts-rk3326/build
 -> /usr/src/linux-headers-6.12.32-lts-rk3326
```

---

# 13. R36S-Armbian RC5のBTF設定確認

## 13.1 なぜ確認するのか

外部Kernel moduleは、実行中Kernelと同じKernel構成を前提にビルドする必要がある。

特に、

```text
CONFIG_DEBUG_INFO_BTF_MODULES
```

はKernel内部の `struct module` の構造にも影響する。

今回のRC5環境では、

- 実行中Kernel
- 配布headers

のBTF設定に差があるため、AX900をDKMS buildする前に整合させる。

---

## 13.2 実Kernel設定

```bash
KVER=$(uname -r)
HDR=/usr/src/linux-headers-$KVER
```

```bash
zcat /proc/config.gz > /tmp/running-kernel.config
```

確認：

```bash
grep '^CONFIG_DEBUG_INFO_BTF' /tmp/running-kernel.config
```

今回：

```text
CONFIG_DEBUG_INFO_BTF=y
CONFIG_DEBUG_INFO_BTF_MODULES=y
```

---

## 13.3 headers側確認

```bash
grep '^CONFIG_DEBUG_INFO_BTF' "$HDR/.config"
```

さらに、

```bash
grep CONFIG_DEBUG_INFO_BTF \
  "$HDR/include/generated/autoconf.h"
```

```bash
grep CONFIG_DEBUG_INFO_BTF \
  "$HDR/include/config/auto.conf"
```

実Kernelと一致していれば変更不要。

一致していない場合のみ補正する。

---

## 13.4 バックアップ

```bash
sudo cp "$HDR/.config" \
        "$HDR/.config.original"
```

```bash
sudo cp "$HDR/include/generated/autoconf.h" \
        "$HDR/include/generated/autoconf.h.original"
```

```bash
sudo cp "$HDR/include/config/auto.conf" \
        "$HDR/include/config/auto.conf.original"
```

---

## 13.5 `.config`

実Kernel設定をコピーする。

```bash
sudo cp /tmp/running-kernel.config "$HDR/.config"
```

---

## 13.6 `autoconf.h`

不足している場合のみ追加する。

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

---

## 13.7 `auto.conf`

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

確認：

```bash
grep CONFIG_DEBUG_INFO_BTF \
  "$HDR/include/generated/autoconf.h"
```

```bash
grep CONFIG_DEBUG_INFO_BTF \
  "$HDR/include/config/auto.conf"
```

---

## 13.8 注意

R36S-Armbianの配布headersは完全なKernel source treeではない。

そのため、

```bash
make prepare
```

や、

```bash
make modules_prepare
```

によってheaders tree全体を再生成しようとしない。

今回必要なのは、

> **外部moduleをコンパイルするときに参照されるKernel設定を、実行中Kernelと一致させること**

である。

---

# 14. AX900ドライバ取得

Driver：

https://github.com/shenmintao/aic8800d80

取得：

```bash
cd ~
git clone https://github.com/shenmintao/aic8800d80.git
cd ~/aic8800d80
```

今回のAX900は `chip_mcu_id=1` なので、

```bash
git switch legacy-mcu1
```

```bash
git pull
```

確認：

```bash
git branch --show-current
```

期待値：

```text
legacy-mcu1
```

---

# 15. AX900ドライバのインストール

## 15.1 インストール中の構成

この時点では、

```text
R36S
  │
  └─ Buffalo WLI-UC-GNM
       └─ 標準rt2800usb
```

のみを接続する。

AX900はまだ接続しない。

理由は、

- SSH接続を安定して維持する
- AX900のfirmware初期化とドライバ導入を分離する
- 給電能力の限られたOTGへ複数の無線アダプタを同時接続しない

ためである。

---

## 15.2 インストール

```bash
cd ~/aic8800d80
```

```bash
chmod +x install.sh
```

```bash
sudo ./install.sh
```

正常時：

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

## 15.3 Bluetooth確認で停止する場合

AX900をまだ接続していない状態では、インストーラ末尾の

```text
Checking for Bluetooth interfaces on combo adapters...
```

でBluetooth deviceを待つ場合がある。

以下まで完了していれば、

```text
DKMS module registered
Wi-Fi kernel module is loaded
Firmware installed
```

主要なインストールは終了している。

この段階なら `Ctrl+C` で終了してよい。

---

# 16. BuffaloからAX900へ差し替える

## 16.1 なぜ電源OFFして交換するのか

USB OTG環境で2つのWi-Fiアダプタを同時に動作させる必要はない。

最終構成そのものを評価するため、

> **BuffaloとAX900を同時接続せず、完全に電源を落として差し替える**

こととする。

シャットダウン：

```bash
sudo poweroff
```

完全に電源が落ちた後、

1. Buffalo WLI-UC-GNMを抜く
2. AX900を挿す
3. R36Sを起動する

---

# 17. AX900のUSB mode switch

AX900は最初からWi-Fi/Bluetooth deviceとして現れるわけではない。

概ね、

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

と再列挙される。

Driver installerが導入する、

- udev rule
- usb_modeswitch設定
- firmware loader

によって自動的に切り替える。

通常運用では手動 `eject` は不要。

---

# 18. AX900 Wi-Fi

Wi-Fi Driver：

```text
aic8800_fdrv
```

GUIではXFCEのネットワークアイコンからSSIDを選択する。

CLI確認：

```bash
nmcli d
```

SSID一覧：

```bash
nmcli device wifi list
```

接続後、

```bash
nmcli d
```

でWi-Fi interfaceが

```text
connected
```

になっていることを確認する。

---

# 19. AX900 Bluetooth

BluetoothはAICSemi専用Bluetooth driverではなく、

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

この状態なら、

```text
Wi-Fi
  → aic8800_fdrv

Bluetooth
  → btusb
```

へ正常にbindされている。

Bluetooth GUIからpairingを行う。

CLI：

```bash
bluetoothctl
```

---

# 20. USB給電・安定性確認

## 20.1 意図

AX900を使用する目的の一つは、

> **Wi-FiとBluetoothを1本へ集約して、R36S OTGのUSB負荷を抑えること**

である。

最終構成でUSBが不安定になっていないことを確認する。

```bash
sudo dmesg \
  | grep -iE 'under|voltage|over.current|error -71|reset high-speed|USB disconnect' \
  | tail -50
```

AX900はmode switchの途中でUSB deviceとして再列挙されるため、起動直後の

```text
USB disconnect
```

数回は正常。

一方、通常利用中に以下が繰り返される場合は注意する。

```text
error -71
reset high-speed USB device
USB disconnect
over-current
undervoltage
```

今回のAX900単独構成では、継続的なUSBエラーや電圧低下の兆候は確認されていない。

---

# 21. 最終USB構成

```bash
lsusb -t
```

最終的に、

```text
btusb
btusb
aic8800_fdrv
```

が同じAX900 deviceにbindされる。

物理的には、

```text
R36S
 │
 └─ OTG
      │
      └─ AX900
           ├─ Wi-Fi
           │    └─ aic8800_fdrv
           │
           └─ Bluetooth
                └─ btusb
```

となる。

Wi-Fi用とBluetooth用に別々のUSB dongleを用意する必要がない。

---

# 22. Buffalo WLI-UC-GNMの位置付け

AX900導入後、Buffaloは通常運用から外す。

ただし、

> **復旧用アダプタとして保管する**

価値が高い。

理由はLinux標準 `rt2800usb` で動作し、AIC8800 driverやDKMSの状態に依存しないためである。

AX900のdriver更新やKernel更新後にWi-Fiが利用できなくなった場合でも、Buffaloを接続してネットワークへ復帰できる。

Wi-Fi deviceが出ない場合：

```bash
sudo modprobe rt2800usb
```

確認：

```bash
nmcli d
```

---

# 23. Kernel更新について

AX900 driverはDKMS登録されている。

確認：

```bash
dkms status
```

通常はKernel更新時に自動再ビルドされる。

ただし、R36Sでは、

- Kernel
- Kernel headers
- DTB
- U-Boot
- BSP

が一般PCより密接に関係している。

そのため、

> **「UbuntuだからとりあえずKernelも最新版へ上げる」という運用は行わない。**

新Kernelへ更新する場合は、

1. R36S対応確認
2. headers有無
3. AX900 DKMS build可否
4. DTB / U-Bootとの整合
5. 起動確認

を行ってから更新する。

---

# 24. AX900 driver更新

Driver更新時：

```bash
cd ~/aic8800d80
```

branch確認：

```bash
git branch --show-current
```

今回：

```text
legacy-mcu1
```

同じ `chip_mcu_id=1` のAX900ではこのbranchを維持する。

---

# 25. NP2kai（任意）

PC的利用の一例として、PC-98 emulatorであるNP2kaiをARM64 native buildできる。

Repository：

https://github.com/AZO234/NP2kai

取得：

```bash
git clone https://github.com/AZO234/NP2kai.git
cd NP2kai
```

SDL2版：

```bash
cmake -S . -B build \
  -DBUILD_SDL=ON \
  -DBUILD_WX=OFF \
  -DBUILD_X=OFF \
  -DUSE_SDL=2
```

Build：

```bash
cmake --build build -j2
```

生成：

```text
build/sdlnp21kai_sdl2
```

R36S / ARM64上でnative実行できる。

これはR36Sが、

> 「ゲーム機用Linuxを動かしているだけ」ではなく、通常のLinux softwareをbuild・実行できる小型PCとして利用可能

であることの確認にもなる。

---

# 26. 完成構成

```text
R36S
│
├─ Armbian / Ubuntu 24.04
│
├─ Linux 6.12.32-lts-rk3326
│
├─ XFCE
│
├─ Fcitx5 + Mozc
│
├─ xvkbd
│
├─ SSH
│
└─ USB OTG
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

# 27. この構成の考え方

今回のキッティングで重視したのは、単純に機能を追加することではない。

### OS

R36S向けに調整されたArmbianを使用し、一般的なLinux PCに近い環境を作る。

### 入力

物理キーボードがなくても `xvkbd` により最低限操作可能にする。

### 日本語

Fcitx5 + Mozcにより通常のLinux desktopとして日本語を扱えるようにする。

### 初期ネットワーク

Linux標準driverで動くBuffaloを利用し、外部driver導入前からSSHとInternet accessを確保する。

### 最終ネットワーク

R36SのOTG給電能力を考慮し、Wi-FiとBluetoothをAX900 1本へ集約する。

### Driver

AX900はDKMS管理とし、ソースからビルド可能な状態を維持する。

### Kernel

R36S独自部分を壊さないことを優先し、不必要なKernel更新を避ける。

---

# 28. キッティング完了状態

最終的に以下を確認できればキッティング完了とする。

- Armbian / XFCE起動
- root filesystemの全容量利用
- 日本語入力
- `xvkbd`による画面キーボード
- SSH接続
- AX900 Wi-Fi接続
- AX900 Bluetooth接続
- `aic8800_fdrv` DKMS登録
- `btusb`正常bind
- USB High-Speed 480M接続
- 継続的なUSBエラーなし
- 電力不足を示す異常ログなし

これによりR36Sを、USB機器を最小限に抑えた**携帯型Linux PC**として利用できる状態とする。