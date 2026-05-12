# Esoteric D-07X Linux USB Audio 修正

Esoteric D-07X DACをLinux（moOde Audio Player）でUSB接続する際、USB High Speed アシンクロナスモード（HS_2）を使用するために必要なカーネルモジュールの修正手順です。

HS_1モードはそのままLinuxで動作しますが、本修正を適用することでHS_2アシンクロナスモードが有効になり、Raspberry Pi 4/5上のmoOde Audio Playerから全サンプリングレート（44.1kHz〜192kHz）の安定した再生が可能になります。

なお、本リポジトリは個人的な検証に基づくものです。動作を保証するものではなく、サポートは提供できません。自己責任でご使用ください。

---

## 対象機器

- **DAC**：Esoteric D-07X
- **ストリーマー**：Raspberry Pi 4B / Raspberry Pi 5B
- **OS / プレーヤーソフト**：moOde Audio Player 10.1.2（Debian Trixie ベース）
- **カーネル**：Linux 6.12.75
  - Raspberry Pi 4B：`6.12.75+rpt-rpi-v8`
  - Raspberry Pi 5B：`6.12.75+rpt-rpi-2712`

※カーネルバージョンが更新された場合は再ビルドが必要です。

---

## 動作確認環境

以下は筆者の確認環境です。他の環境での動作は未確認ですが、同様の構成であれば動作すると思われます。

- **ストリーマー**：
  - Raspberry Pi 4B 4GB：ファンレスアルミケース（サーマルパッド使用）
  - Raspberry Pi 5B 4GB：公式アクティブクーラー付き金属ケース
- **電源**：USB-C PD対応アダプター 27W
- **NAS**：アイ・オー・データ Soundgenic HDL-RA2HF
- **ネットワーク**：有線LAN
- **コントロール**：fidata Music App（iOS）/ LUMIN（iOS）
- **接続プロトコル**：OpenHome
- **USBケーブル**：D-07X〜Raspberry Pi間

---

## 問題の背景

Esoteric D-07Xは3つのUSB接続モードを持ちます。

- **NORM**：USB FULL SPEEDモードで接続します。最大の対応サンプリング周波数は96kHzまでです。Linuxで追加修正なしに動作します
- **HS_1**：USB HIGH SPEEDモードで接続します。最大の対応サンプリング周波数は192kHzまでです。Linuxで追加修正なしに動作しますが、HS_2モードよりジッター対策が弱くなります
- **HS_2**：USB High Speed アシンクロナスモードで接続します。最大の対応サンプリング周波数は192kHzまでです。ジッター対策が強固になりますが、Linux標準カーネルでは正常に動作しません

HS_2モードでLinux標準カーネルを使用した場合、以下の問題が発生します：

- 高サンプリングレート（88.2kHz以上）での再生時にノイズ・歪みが発生する
- 曲の切り替わり時にALSA（LinuxのオーディオシステムがD-07Xを管理している仕組み）がタイムアウトエラーを起こす
- 長時間再生後に再生が停止し、ALSAによるD-07Xの管理が応答不能になる

> 注：44.1kHz・48kHzについては修正前の長時間動作は未確認ですが、修正後は24時間以上の安定動作を確認しています。

これらの問題はD-07XのUSBディスクリプタの特性と、LinuxカーネルのUSBオーディオドライバーの処理との不整合によるものです。本修正はこれらの問題を解決します。

---

## 修正内容

以下の3つのカーネルソースファイルを修正します。

### sound/usb/quirks.c（2箇所）

**1箇所目：tenor_fb_quirkへの追加**

D-07XのUSBフィードバック値の補正処理に追加します。同系統のチップ（TE8802L）を使用するTEAC UD-H01が既に登録されており、D-07Xも同様の処理が必要です。

**2箇所目：DEVICE_FLGエントリの追加**

D-07X専用の動作フラグを追加します。

- `QUIRK_FLAG_IFACE_DELAY`：インターフェース初期化時の待機
- `QUIRK_FLAG_FORCE_IFACE_RESET`：インターフェースのリセット強制
- `QUIRK_FLAG_IGNORE_CLOCK_SOURCE`：クロックソース検証のスキップ

### sound/usb/endpoint.c（1箇所・Raspberry Pi 4Bのみ）

D-07XのUSBコントローラー（VL805）固有の問題に対応するため、フィードバックエンドポイントの処理にD-07X専用の判定を追加します。

### sound/usb/clock.c（1箇所）

曲の切り替わり時に発生するクロック有効性チェックのタイムアウト問題を解決します。`QUIRK_FLAG_IGNORE_CLOCK_SOURCE`フラグをクロック有効性チェックにも適用されるよう修正します。

---

## インストール手順

### 前提条件
- moOde Audio Playerがインストール済みで起動していること
- SSHで接続できること
- インターネットに接続されていること

### 1. システムの更新

```bash
sudo apt update && sudo apt upgrade -y
```

完了後、再起動します：

```bash
sudo reboot
```

### 2. カーネルバージョンの確認

再接続後に確認します：

```bash
uname -r
```

- Raspberry Pi 4B：`6.12.75+rpt-rpi-v8`
- Raspberry Pi 5B：`6.12.75+rpt-rpi-2712`

と表示されれば正常です。

### 3. ビルドツールとgitのインストール

```bash
sudo apt install -y git build-essential bc bison flex libssl-dev libelf-dev
```

### 4. カーネルソースの取得

```bash
cd ~
git clone --depth 1 --branch rpi-6.12.y --no-checkout https://github.com/raspberrypi/linux.git rpi-linux
cd rpi-linux
git sparse-checkout init
git sparse-checkout set sound/usb
git checkout
```

### 5. quirks.cの修正

Pythonインタラクティブモードで修正します：

```bash
python3
```

以下を1行ずつ実行してください：

```python
f = open('sound/usb/quirks.c', 'r')
content = f.read()
f.close()
```

```python
old1 = '\tif ((ep->chip->usb_id == USB_ID(0x0644, 0x8038) ||  /* TEAC UD-H01 */'
new1 = '\tif ((ep->chip->usb_id == USB_ID(0x0644, 0x8038) ||  /* TEAC UD-H01 */\n\t    ep->chip->usb_id == USB_ID(0x0644, 0x802c) ||  /* Esoteric D-07X */'
print(old1 in content)
```

`True`と表示されたら続けます：

```python
content = content.replace(old1, new1)
```

```python
old2 = 'DEVICE_FLG(0x0644, 0x8044, /* Esoteric D-05X */'
new2 = 'DEVICE_FLG(0x0644, 0x802c, /* Esoteric D-07X */\n\t   QUIRK_FLAG_IFACE_DELAY | QUIRK_FLAG_FORCE_IFACE_RESET |\n\t   QUIRK_FLAG_IGNORE_CLOCK_SOURCE),\n' + old2
print(old2 in content)
```

`True`と表示されたら続けます：

```python
content = content.replace(old2, new2)
f = open('sound/usb/quirks.c', 'w')
f.write(content)
f.close()
exit()
```

修正を確認します：

```bash
grep -n "D-07X" sound/usb/quirks.c
```

以下のように2行表示されれば正常です：
1953:     ep->chip->usb_id == USB_ID(0x0644, 0x802c) ||  /* Esoteric D-07X /
2220: DEVICE_FLG(0x0644, 0x802c, / Esoteric D-07X */

### 6. clock.cの修正

```bash
nano +314 ~/rpi-linux/sound/usb/clock.c
```

以下の部分を：
```c
if (validate && !uac_clock_source_is_valid(chip, fmt,
    entity_id)) {
```

以下に変更してください：
```c
if (validate && !(chip->quirk_flags & QUIRK_FLAG_IGNORE_CLOCK_SOURCE) &&
    !uac_clock_source_is_valid(chip, fmt,
    entity_id)) {
```

`Ctrl+O` → Enter → `Ctrl+X` で保存します。

### 7. endpoint.cの修正（Raspberry Pi 4Bのみ）

> **注：Raspberry Pi 5Bではこの手順は不要です**

```bash
nano +1878 ~/rpi-linux/sound/usb/endpoint.c
```

以下の部分を：
```c
if (unlikely(sender->tenor_fb_quirk)) {
```

以下に変更してください：
```c
if (unlikely(sender->tenor_fb_quirk ||
    sender->chip->usb_id == 0x0644802c)) {
```

`Ctrl+O` → Enter → `Ctrl+X` で保存します。

### 8. ビルド

**Raspberry Pi 4Bの場合：**

```bash
cd ~/rpi-linux
make -C /usr/src/linux-headers-6.12.75+rpt-rpi-v8 M=$(pwd)/sound/usb modules 2>&1 | tail -10
```

**Raspberry Pi 5Bの場合：**

```bash
cd ~/rpi-linux
make -C /usr/src/linux-headers-6.12.75+rpt-rpi-2712 M=$(pwd)/sound/usb modules 2>&1 | tail -10
```

最後の行に以下のように表示されれば成功です：
make: Leaving directory '/usr/src/linux-headers-6.12.75+rpt-rpi-xxxx'

### 9. インストール

> **重要：** Raspberry Pi OSでは圧縮済みモジュール（`.ko.xz`）が優先されてロードされます。必ず以下の手順で削除してからインストールしてください。削除しないと修正が反映されません。

**Raspberry Pi 4B / 5B共通：**

```bash
sudo rm /lib/modules/$(uname -r)/kernel/sound/usb/snd-usb-audio.ko.xz
sudo cp sound/usb/snd-usb-audio.ko /lib/modules/$(uname -r)/kernel/sound/usb/
sudo depmod -a
sudo reboot
```

---

## moOde設定（手順9完了後に実施）

### 10. D-07X本体のUSB設定

D-07X本体のMENUボタンでUSB入力設定を**HS_2**に変更してください。

1. MENUボタンを繰り返し押して`USB>***`を表示
2. VOLUMEボタン（+/-）で`HS_2`を選択
3. 入力切換ボタンを押すか10秒以上放置すると設定モードが終了します

### 11. D-07XをUSBで接続

D-07XとRaspberry PiをUSBケーブルで接続してください。USB2.0/3.0どちらの端子でも動作します。

### 12. moOdeのオーディオデバイス設定

1. ブラウザで`http://moode.local`またはIPアドレスにアクセス
2. 右上の「m」マーク → **Configure** → **Audio**
3. **Output device** で「ESOTERIC USB AUDIO DEVICE」を選択
4. **Volume type** を「Fixed」に設定
5. **Save** をクリック

### OpenHomeを使用する場合の追加設定

1. 右上の「m」マーク → **Configure** → **Audio** → **Renderers**
2. **UPnP Client for MPD**サービスをONに設定
3. **EDIT**をクリック → **General** → **Service type**を「OpenHome」に変更
4. **Save**をクリック

---

## 動作確認

D-07Xが認識されているか確認します：

```bash
aplay -l | grep ESOTERIC
```

以下のように表示されれば正常です：
card X: DEVICE [ESOTERIC USB AUDIO DEVICE], device 0: USB Audio [USB Audio]

再生中に転送周波数を確認します（96kHz再生時の例）：

```bash
cat /proc/asound/DEVICE/stream0
```

`Momentary freq`が再生周波数に近い値で安定していれば正常動作しています。

### USBオートサスペンドの無効化

LinuxはUSBデバイスを一定時間使用しないと省電力のためにサスペンド状態にします。長時間再生時の意図しない停止を防ぐため、以下の設定を行います：

```bash
sudo nano /etc/udev/rules.d/99-usb-autosuspend.rules
```

以下の内容を入力して保存します：
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="0644", ATTR{idProduct}=="802c", ATTR{power/control}="on"

`Ctrl+O` → Enter → `Ctrl+X` で保存後、設定を反映します：

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## テスト結果

以下の環境で全サンプリングレートの24時間連続再生テストを実施し、正常動作を確認しました。

※転送周波数の表記は測定中に観測された最小値/最大値です。目標値より若干高めになるのはASYNCモードの正常な動作です。

### Raspberry Pi 4B

| サンプリングレート | 転送周波数 | テスト時間 | 結果 |
|------------|--------|--------|-----|
| 44.1kHz | 44104/44105Hz | 24時間以上 | ✅ 正常 |
| 48kHz | 48004/48006Hz | 24時間以上 | ✅ 正常 |
| 88.2kHz | 88209/88211Hz | 24時間以上 | ✅ 正常 |
| 96kHz | 96010/96012Hz | 24時間以上 | ✅ 正常 |
| 176.4kHz | 176420/176422Hz | 24時間以上 | ✅ 正常 |
| 192kHz | 192023/192025Hz | 24時間以上 | ✅ 正常 |

### Raspberry Pi 5B

| サンプリングレート | 転送周波数 | テスト時間 | 結果 |
|------------|--------|--------|-----|
| 44.1kHz | 44100/44102Hz | 24時間以上 | ✅ 正常 |
| 48kHz | 48000/48002Hz | 24時間以上 | ✅ 正常 |
| 88.2kHz | 88201/88203Hz | 24時間以上 | ✅ 正常 |
| 96kHz | 96002/96004Hz | 24時間以上 | ✅ 正常 |
| 176.4kHz | 176406/176408Hz | 24時間以上 | ✅ 正常 |
| 192kHz | 192008/192010Hz | 24時間以上 | ✅ 正常 |

### 考察

転送周波数が目標値より若干高めになるのはASYNCモードの正常な動作です。D-07X内部のバッファ管理によりバッファ枯渇を防ぐために少し多めにデータを要求しています。Pi 5の方がより目標値に近い転送精度を示しています。

---

## 注意事項

### カーネルアップデート時の再ビルドについて

Raspberry Pi OSのカーネルがアップデートされた場合、インストールしたモジュールは無効になり本修正の効果が失われます。カーネルの自動アップデートを停止しておくか、アップデート後に手順4〜9を再実施して再ビルドするかが必要となります。

カーネルバージョンが変わったかどうかは以下で確認できます：

```bash
uname -r
```

### .xz圧縮ファイルの削除について

Raspberry Pi OSではカーネルモジュールの圧縮版（`.ko.xz`）が優先されてロードされます。インストール時に必ず削除してください。削除しないと修正が反映されません。

### D-07X本体の設定について

USB入力設定が**HS_2**以外（NORM、HS_1）になっている場合、本修正の効果は得られません。必ずHS_2に設定してください。

### サポートについて

本リポジトリは個人的な検証に基づくものです。動作を保証するものではなく、サポートは提供できません。自己責任でご使用ください。

---

## クレジット

本修正の調査・実装・検証はAnthropic社のAIアシスタント「Claude」のサポートのもとで行いました。同様の課題を抱えている方はClaudeに相談することで解決の糸口が見つかるかもしれません。
