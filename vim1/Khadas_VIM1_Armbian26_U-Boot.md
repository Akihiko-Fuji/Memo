# Khadas VIM1 V1.4：Armbian 26.8.11起動・Mainline U-BootのeMMC更新記録

- **記録日**：2026年11月20日
- **対象**：Khadas VIM1 V1.4（Amlogic S905X / GXL、DDR3 2 GiB、eMMC約7.3 GiB）
- **目的**：既存eMMCの古いU-Bootでは起動できない新しいArmbianをmicroSDから起動し、最終的には電源投入・通常再起動時にFunctionボタン操作なしで起動する。HDMI接続のX Windowを720×480にする。
- **最終結果**：**eMMC上のMainline U-Boot 2021.07が正常に起動し、microSD上のArmbian 26.8.11 Resolute / Linux 6.18.57-current-meson64のログインプロンプトまで到達。** 720×480のカーネル設定・Xorgの手動切り替えにも成功。

> [!IMPORTANT]
> この文書は**実際のVIM1 V1.4の検証記録**。同種のボードでもブート領域、MMC番号、OS構成が同じとは限らない。以下の`dd`操作はeMMCのブートローダーを直接書き換えるため、実行前にデバイスを必ず識別すること。

## 1. 構成と確定したデバイス名

| 項目 | 値・状態 |
|---|---|
| SoC / 基板 | Amlogic S905X (GXL) / Khadas VIM1 V1.4 |
| DRAM | 2 GiB（DDR3、Rank0+1） |
| 既存ブートローダー | Vendor U-Boot `2015.01 (Jul 25 2022)`、当初はeMMCに存在 |
| 新ブートローダー | Khadas Mainline U-Boot `2021.07 (Sep 19 2025)`、公式リリース `0.17.3` |
| 新OS | Armbian `26.8.11` Resolute / Linux `6.18.57-current-meson64`、XFCEデスクトップ版 |
| microSD | `/dev/mmcblk0`、57.6 GiB、`/dev/mmcblk0p1` (ext4) に `/` |
| eMMC | `/dev/mmcblk1`、7.3 GiB（`7818182656` bytes） |
| eMMC独立ブート領域 | `/dev/mmcblk1boot0`、`/dev/mmcblk1boot1`、各4 MiB |
| 作業用Raspberry Pi | オンライン。公式バイナリの取得とUSBメモリーへのコピーに使用 |
| VIM1 | オフライン。UARTコンソールとmicroSD経由で操作 |
| ファイル転送 | USBメモリー（VIM1では`/dev/sda1`、vfat） |

起動中のArmbianでの実測：

```console
$ findmnt -no SOURCE /
/dev/mmcblk0p1
$ cat /sys/block/mmcblk0/device/type
SD
$ cat /sys/block/mmcblk1/device/type
MMC
$ sudo blockdev --getsize64 /dev/mmcblk1
7818182656
$ df -h /home/pi
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p1   57G  4.7G   50G   9% /
```

**注意：Linuxの`/dev/mmcblk*`番号とU-Bootが表示する`mmc 0:1`、`mmc 1:1`の番号は一致するとは限らない。書き込み対象の確認にはLinux側の`/sys/block/.../device/type`を使用した。**

## 2. 最初の障害：Vendor U-Bootでは新しいArmbianを起動できない

使用したイメージ：

```text
Armbian_26.8.11_Khadas-vim1_resolute_current_6.18.57_xfce_desktop.img.xz
```

### 2.1 ext4の互換性

当初、古いU-BootがmicroSDのext4を読み込めず、次のメッセージが発生した。

```text
Failed to mount ext2 filesystem...
** Unrecognized filesystem type **
```

検証ではmicroSDのext4に新しい機能（`64bit`、`metadata_csum`、`metadata_csum_seed`）が有効なことが問題となった。microSDのパーティションを互換性重視のext4として再作成し、必要なデータを戻した。

```text
-O ^64bit,^metadata_csum,^metadata_csum_seed
```

この対応後、Vendor U-Bootから次の各ファイルの読み込みは成功した。

```console
kvim# ext4load mmc 0:1 0x18000000 /boot/uInitrd
kvim# ext4load mmc 0:1 0x34000000 /boot/Image
kvim# ext4load mmc 0:1 0x20000000 /boot/dtb/amlogic/meson-gxl-s905x-khadas-vim.dtb
```

読み込み確認済みのサイズ：`uInitrd` 33,126,842 bytes、`Image` 37,689,856 bytes、DTB 42,212 bytes。

rootfsのUUIDは次のまま維持した。

```text
6a597149-e131-41c8-806d-078ea2a66162
```

> ext4の再作成は**今回の事前検証で実施した対応**であり、Mainline U-Boot導入後も必ず必要になるという意味ではない。

### 2.2 `booti`で同期例外

旧U-Bootでカーネル起動を試すと、Linux起動前に例外が発生した。

```console
kvim# booti 0x34000000 0x18000000 0x20000000
"Synchronous Abort" handler, esr 0x96000210
ELR: 77e9f1f4
LR:  77e9f1e8
Resetting CPU ...
```

途中で`[rsvmem] fdt get prop fail.`や`FDT_ERR_NOTFOUND`も確認。DTBをRAM上で調整し、`/reserved-memory/linux,secmon`を追加して`rsvmem check`を通したが、**`booti`の同期例外は解消しなかった**。このため、Vendor U-Bootに対する互換性回避を継続せず、Mainline U-Bootに移行した。

## 3. USB Burning Toolによる更新は失敗

最初はRaspberry PiからUSB-C経由でVIM1のeMMCブートローダーを書き換えることを試した。

### 3.1 USB Upgrade Modeへの移行

Vendor U-BootのUARTから実行：

```console
kvim# printenv update
update=run usb_burning; run sdc_burning; if mmcinfo; then run recovery_from_sdcard;fi;if usb start 0; then run recovery_from_udisk;fi;run recovery_from_flash;
kvim# run update
InUsbBurn
[MSG]sof
Set Addr 2
Get DT cfg
Get DT cfg
set CFG
```

Raspberry Piではデバイスを認識した。

```console
$ lsusb -d 1b8e:c003
Bus 001 Device 005: ID 1b8e:c003 Amlogic, Inc.
$ sudo /home/pi/utils/aml-flash-tool/tools/linux-arm/update identify 7
AmlUsbIdentifyHost
This firmware version is 0-7-0-16-0-0-0
```

### 3.2 DDR初期化用データのUSB転送エラー

公式`khadas/utils`の`flash-tool`で、`--parts=bootloader`および診断用`--parts=none --debug`を試したが、DDR初期化データ転送で停止。

```text
Unpacking image [OK]
Initializing ddr
Command -> .../update cwr .../DDR.USB 0xd9000000
[LUSB]IOCTL_WRITE_MEM_Handler ret=-71
error_msg=error sending control message: Protocol error
ERR: write data to device failed
[KO]
```

Raspberry Pi側`dmesg`にも次のエラーが記録された。

```text
usbfs: USBDEVFS_CONTROL failed cmd update rqt 64 rq 1 len 64 ret -71
```

- `-71`はUSBのプロトコルエラー（`EPROTO`）。
- `awk`のエスケープ警告も出ていたが、イメージ展開や設定値抽出は完了しており、主要因とは判断しなかった。
- eMMCへの書き込みステップには到達していないと判断。
- **USB書き込み経路の問題は未解決のまま**、別経路で成功した。

### 3.3 使用したUSB書き込み向けパッケージ

```text
VIM1.uboot-mainline.emmc.aml.img
SHA-256: bbf96ace12a12fd23af17ab7942f54eced47274d367948742b8d4e436b75c698
```

これは**USB Burning Tool向けの`.aml.img`パッケージ**であり、eMMCへ`dd`するバイナリではない。直接書き込みには後述の`VIM1.u-boot.sd.bin`を使った。

## 4. microSD上のMainline U-BootでArmbian起動に成功

VIM1 V1.4のFunctionボタン操作によってmicroSD起動を強制したところ、次のログを確認した。

```text
Load fip header from SD
Load bl30 from SD
Load bl31 from SD
Load bl33 from SD
...
U-Boot 2021.07 (Sep 19 2025 - 11:12:38 +0700) khadas-vim
Model: Khadas VIM
SoC: Amlogic Meson GXL (S905X)
DRAM: 2 GiB
...
Found U-Boot script /boot/boot.cmd
...
Starting kernel ...
...
Armbian 26.8.11 Resolute ttyAML0
khadas-vim1 login:
```

初回ログインはArmbianの初期認証情報を使用し、その後一般ユーザー`pi`で操作した。Armbian上で`Linux 6.18.57-current-meson64`を確認した。

ただしこの時点では、**通常の再起動で旧eMMC側のU-Boot 2015.01に戻る**ため、SD起動を毎回再現するにはFunctionボタンの操作が必要だった。これがeMMCブートローダー更新に進んだ理由。

## 5. HDMI / X Windowを720×480に変更

### 5.1 カーネル側の指定

microSD上の`/boot/armbianEnv.txt`の`extraargs`に次を追加した。

```ini
extraargs=video=HDMI-A-1:720x480@60
```

既存の`extraargs`がある場合は**同じ行へ追記**し、既存オプションは維持する。後のU-Boot起動ログで反映を確認。

```text
Mainline bootargs: ... video=HDMI-A-1:720x480@60 ...
```

Linuxのフレームバッファは次のとおりになった。

```console
$ cat /sys/class/graphics/fb0/virtual_size
720,480
```

### 5.2 Xorgは別途1920×1080で起動していた

実際のHDMI画面は高解像度のままで文字が読みにくかった。UART上で普通に`xrandr`を実行すると`Can't open display`となったが、XorgプロセスがLightDMの認証ファイルを使用していたことを確認した。

```text
/usr/lib/xorg/Xorg -core :0 -seat seat0 -auth /var/run/lightdm/root/:0 ...
```

認証ファイルを指定してXorgの状態を取得：

```bash
sudo env DISPLAY=:0 \
  XAUTHORITY=/var/run/lightdm/root/:0 \
  xrandr --query
```

結果：

```text
Screen 0: current 1920 x 1080
HDMI-1 connected primary 1920x1080+0+0
   1920x1080  60.00*+
   ...
   720x480    59.94
```

**カーネル側の解像度とXorg側の解像度は別**で、Xorg側の変更が必要だった。

### 5.3 Xorgを720×480へ変更（成功を確認）

```bash
sudo env DISPLAY=:0 \
  XAUTHORITY=/var/run/lightdm/root/:0 \
  xrandr --output HDMI-1 --mode 720x480 --rate 59.94
```

これによりHDMI画面が見やすい720×480表示になった。

LightDM起動時も設定するため、次の永続化手順を案内した。

```bash
sudo tee /usr/local/bin/vim1-display-480p.sh >/dev/null <<'SH'
#!/bin/sh
/usr/bin/xrandr --output HDMI-1 --mode 720x480 --rate 59.94
SH
sudo chmod +x /usr/local/bin/vim1-display-480p.sh

sudo mkdir -p /etc/lightdm/lightdm.conf.d
sudo tee /etc/lightdm/lightdm.conf.d/50-vim1-display.conf >/dev/null <<'CONF'
[Seat:*]
display-setup-script=/usr/local/bin/vim1-display-480p.sh
CONF
```

> **検証範囲の区別**：`xrandr`の即時切り替え成功は報告済み。最後に提供された再起動UARTログではカーネル引数への720×480設定反映は確認できるが、**再起動後のXorg実解像度／LightDM設定の永続化結果は`xrandr --current`で再確認する余地がある**。

## 6. eMMC上のU-BootをMainlineへ更新

### 6.1 書き込み用バイナリの取得

Raspberry Pi（オンライン）で、Khadas公式リリース`0.17.3`から取得。

```bash
cd /home/pi/Prodtrac
wget https://github.com/khadas/khadas-uboot/releases/download/0.17.3/VIM1.u-boot.sd.bin
sha256sum VIM1.u-boot.sd.bin
```

```text
bb4ea91fc445bf305e82f8447c868e6ee9ef8b8e3e83f35905cfb3956b8a96d1  VIM1.u-boot.sd.bin
```

USBメモリーでVIM1へ転送（VIM1側の例）。

```bash
sudo mkdir -p /mnt/usb
sudo mount -o ro /dev/sda1 /mnt/usb
cp /mnt/usb/VIM1.u-boot.sd.bin /home/pi/
sudo umount /mnt/usb
sha256sum /home/pi/VIM1.u-boot.sd.bin
```

VIM1側でもSHA-256一致を確認した。

### 6.2 書き込み前確認

```console
$ cat /sys/block/mmcblk0/device/type
SD
$ cat /sys/block/mmcblk1/device/type
MMC
$ findmnt -no SOURCE /
/dev/mmcblk0p1
$ sha256sum /home/pi/VIM1.u-boot.sd.bin
bb4ea91fc445bf305e82f8447c868e6ee9ef8b8e3e83f35905cfb3956b8a96d1  /home/pi/VIM1.u-boot.sd.bin
```

**書き込み先は必ず`/dev/mmcblk1`（eMMC）。`/dev/mmcblk0`（稼働中のmicroSD）ではない。**

### 6.3 バックアップに関する判断と注意

microSDの空き容量は約50GBあったため、eMMC約7.3GBの全体バックアップを**USBメモリーなしでmicroSD上へ保存可能**だった。しかし、今回の作業ではユーザー判断で**バックアップを省略**した。

これは再現時の推奨手順ではない。復旧能力を高めるなら、書き込み前に以下を実施すること。

```bash
sudo mkdir -p /home/pi/vim1-emmc-backup
sudo dd if=/dev/mmcblk1 \
  of=/home/pi/vim1-emmc-backup/emmc-full.img \
  bs=4M iflag=fullblock status=progress conv=fsync
sudo dd if=/dev/mmcblk1boot0 \
  of=/home/pi/vim1-emmc-backup/emmc-boot0.img bs=1M status=progress
sudo dd if=/dev/mmcblk1boot1 \
  of=/home/pi/vim1-emmc-backup/emmc-boot1.img bs=1M status=progress
sudo cmp /dev/mmcblk1 /home/pi/vim1-emmc-backup/emmc-full.img
```

> **重要**：上記は推奨されるバックアップ例であり、**今回実際に取得したという記録ではない**。バックアップなしに起動領域を更新すると、旧Ubuntuのブート環境を元に戻せない可能性がある。

### 6.4 実際に成功した`dd`書き込み

Khadas公式の`VIMx sd / mmc / emmc uboot`手順に沿い、先頭442バイトと512バイト以降を分けて書き込んだ。これはMBRのパーティション表部分（446〜509バイトなど）を避けるための形式である。ただし、Amlogic独自パーティションの完全保持や旧Ubuntuの起動互換性までは保証しない。

```bash
# eMMC先頭442バイト
sudo dd if=/home/pi/VIM1.u-boot.sd.bin \
  of=/dev/mmcblk1 \
  bs=442 count=1 \
  conv=fsync,notrunc

# 512バイト目以降（元ファイルのセクター1以降）
sudo dd if=/home/pi/VIM1.u-boot.sd.bin \
  of=/dev/mmcblk1 \
  bs=512 skip=1 seek=1 \
  conv=fsync,notrunc

sync
```

実行結果：

```text
1+0 records in
1+0 records out
442 bytes copied, 0.00590233 s, 88 kB/s
2663+0 records in
2663+0 records out
1363456 bytes (1.4 MB, 1.3 MiB) copied, 0.100115 s, 14 MB/s
```

### 6.5 読み戻し照合（成功）

```bash
IMG=/home/pi/VIM1.u-boot.sd.bin
SIZE=$(stat -c%s "$IMG")

sudo cmp -n 442 "$IMG" /dev/mmcblk1 && \
sudo cmp -n "$((SIZE - 512))" -i 512:512 "$IMG" /dev/mmcblk1 && \
echo "U-Boot verification OK"
```

```text
U-Boot verification OK
```

つまり、**指定したeMMC領域の内容がファイルと一致**した。ただし、この時点ではBootROMが起動するかは未検証だった。

## 7. 最終検証：eMMCから新U-Boot、microSDからArmbianを自動起動

microSDを挿入したまま`sudo reboot`を実施し、**Functionボタンを押さずに起動成功**。

### 7.1 eMMCからブートローダーを読み込んだ

```text
Load fip header from eMMC, src: 0x0000c200, des: 0x01400000, size: 0x00004000
Load bl30 from eMMC, src: 0x00010200, des: 0x013c0000, size: 0x0000d600
Load bl31 from eMMC, src: 0x00020200, des: 0x05100000, size: 0x0003b600
Load bl33 from eMMC, src: 0x0005c200, des: 0x01000000, size: 0x000bfa00
...
U-Boot 2021.07 (Sep 19 2025 - 11:12:38 +0700) khadas-vim
Model: Khadas VIM
SoC: Amlogic Meson GXL (S905X)
DRAM: 2 GiB
```

ここが最大の成功判定ポイント。**U-Bootの起動元はeMMCで、起動したのは新しいMainline U-Boot**である。

### 7.2 microSD上のArmbianを見つけた

```text
Scanning mmc 1:1...
Found U-Boot script /boot/boot.cmd
Current fdtfile after armbianEnv: amlogic/meson-gxl-s905x-khadas-vim.dtb
Mainline bootargs: root=UUID=6a597149-e131-41c8-806d-078ea2a66162 ... video=HDMI-A-1:720x480@60 ...
33126842 bytes read
37689856 bytes read
42212 bytes read
...
Starting kernel ...
```

UARTの`mmc 1:1`は**U-Boot側の番号**。Linux側では起動中のmicroSDが`/dev/mmcblk0p1`になっていることを別途確認済み。

### 7.3 Linuxログイン画面まで到達

```text
[   11.961767] lima d00c0000.gpu: error -ENODEV: _opp_set_regulators: no regulator (mali) found
[   12.295151] brcmfmac: brcmf_fw_alloc_request: using brcm/brcmfmac43430-sdio for chip BCM43430/1
[   12.581176] brcmfmac: brcmf_c_preinit_dcmds: Firmware: BCM43430/1 ...

Armbian 26.8.11 Resolute ttyAML0
khadas-vim1 login:
```

GPUの`lima`エラー、Wi-Fiの`txcap_blob`に関する警告、U-Bootの`bad CRC, using default environment`などは記録に残るが、**今回のログではログインプロンプト到達を妨げていない**。GUI表示・GPUアクセラレーション・Wi-Fi通信性能の全面的な正常性まで証明するものではない。

## 8. 最終状態・残作業

| 確認項目 | 状態 |
|---|---|
| 旧U-Bootによる`booti`同期例外の回避 | **達成**（Mainline U-Bootへ移行） |
| microSD上のArmbian 26.8.11の起動 | **達成** |
| eMMC上のMainline U-Boot更新 | **達成**（書き込み・読み戻し・eMMCからの起動を確認） |
| Functionボタン操作なしの再起動 | **達成** |
| カーネル引数の720×480設定 | **確認済み** |
| Xorgの720×480即時切り替え | **確認済み** |
| 再起動後のXorg解像度の永続化 | **要最終確認**（起動ログだけでは判断できない） |
| eMMCの旧Ubuntuとの互換性 | **未確認**（更新前のバックアップも未取得） |
| USB Burning Toolの`-71`転送エラー | **未解決**（通常運用では回避） |
| Wi-Fi接続、GUIアプリ、長時間安定性 | **別途評価** |

再起動後の画面を検証する場合：

```bash
sudo env DISPLAY=:0 XAUTHORITY=/var/run/lightdm/root/:0 xrandr --current
```

出力の`720x480`に`*`が付けば、Xorgの実解像度も720×480である。

## 9. 再現時に避けること

1. **`.aml.img`を`dd`でeMMCへ書かない**。今回直接書き込んだのは`VIM1.u-boot.sd.bin`。
2. **microSD（`mmcblk0`）へ誤書き込みしない**。Linuxで`SD`と`MMC`を識別する。
3. **バックアップを省略した手順を一般化しない**。今回成功したが、再現時はeMMC全体の事前保存を推奨する。
4. USB Burning Toolの`Initializing ddr [KO]`をDDRハードウェア故障と即断しない。今回の直接エラーは`update cwr`のUSB制御転送`-71`だった。
5. **`fb0/virtual_size=720,480`だけでXorgの実解像度が720×480だと判断しない**。Xorg側は別に1920×1080だった。
6. `mmc`番号はU-BootとLinuxで異なりうるため、起動ログの番号をそのまま`dd`先に流用しない。

## 10. 参考資料

- [Khadas Mainline U-Boot 0.17.3（公式リリース、バイナリ・SHA-256・書き込み例）](https://github.com/khadas/khadas-uboot/releases/tag/0.17.3)
- [Khadas utils / aml-flash-tool（USB書き込みスクリプト）](https://github.com/khadas/utils/tree/master/aml-flash-tool)
- [Khadas公式ドキュメント：VIM1](https://docs.khadas.com/products/sbc/vim1/)

---

**結論**：本件では、Vendor U-Boot 2015.01によるArmbianカーネル起動不能を、Khadas Mainline U-Boot 0.17.3への更新で解消した。最初はmicroSDのMainline U-BootからArmbianを起動してソフトウェア互換性を確認し、次に稼働中のArmbianからeMMC上のU-Bootを直接更新した。最終ログにより、**eMMCからMainline U-Boot 2021.07 → microSD上のArmbian 26.8.11**という通常再起動経路が成立したことを確認した。
