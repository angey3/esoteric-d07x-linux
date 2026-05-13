# Esoteric D-07X Linux USB Audio Fix

[日本語版はこちら / Japanese version](README.md)

This repository provides the necessary kernel module patches to enable USB High Speed Asynchronous mode (HS_2) for the Esoteric D-07X DAC when used with Linux (moOde Audio Player).

While HS_1 mode works on Linux without any modifications, applying this fix enables HS_2 asynchronous mode, allowing stable playback at all sample rates (44.1kHz to 192kHz) on Raspberry Pi 4/5 with moOde Audio Player.

> **Note:** This repository is based on personal testing. No warranty is provided and no support can be offered. Use at your own risk.

---

## Target Hardware

- **DAC**: Esoteric D-07X
- **Streamer**: Raspberry Pi 4B / Raspberry Pi 5B
- **OS / Player**: moOde Audio Player 10.1.2 (Debian Trixie based)
- **Kernel**: Linux 6.12.75
  - Raspberry Pi 4B: `6.12.75+rpt-rpi-v8`
  - Raspberry Pi 5B: `6.12.75+rpt-rpi-2712`

> If the kernel version is updated, a rebuild will be required.

---

## Test Environment

The following is the author's test environment. Operation in other environments has not been verified, but similar configurations should work.

- **Streamer**:
  - Raspberry Pi 4B 4GB: Fanless aluminum case (with thermal pads)
  - Raspberry Pi 5B 4GB: Official active cooler metal case
- **Power supply**: USB-C PD adapter 27W
- **NAS**: I-O DATA Soundgenic HDL-RA2HF
- **Network**: Wired LAN
- **Control app**: fidata Music App (iOS) / LUMIN (iOS)
- **Protocol**: OpenHome
- **USB cable**: Between D-07X and Raspberry Pi

---

## Background

The Esoteric D-07X has three USB connection modes:

- **NORM**: USB FULL SPEED mode. Maximum supported sample rate is 96kHz. Works on Linux without modification
- **HS_1**: USB HIGH SPEED mode. Maximum supported sample rate is 192kHz. Works on Linux without modification, but provides less jitter reduction than HS_2
- **HS_2**: USB High Speed Asynchronous mode. Maximum supported sample rate is 192kHz. Provides superior jitter reduction, but does not work correctly with the standard Linux kernel

When using HS_2 mode with the standard Linux kernel, the following problems occur:

- Noise and distortion during playback at high sample rates (88.2kHz and above)
- ALSA (the Linux audio subsystem managing the D-07X) timeout errors during track changes
- Playback stops after extended use and ALSA becomes unresponsive

> Note: Long-term operation at 44.1kHz and 48kHz in HS_2 mode was not tested before applying this fix. However, stable operation for 24+ hours has been confirmed after applying the fix.

These issues are caused by incompatibilities between the D-07X's USB descriptor characteristics and the Linux kernel's USB audio driver processing. This fix resolves these issues.

---

## What Is Modified

Three kernel source files are modified:

### sound/usb/quirks.c (2 changes)

**Change 1: Add to tenor_fb_quirk**

Adds D-07X to the USB feedback value correction processing. The TEAC UD-H01, which uses the same chipset (TE8802L), is already registered and the D-07X requires the same processing.

**Change 2: Add DEVICE_FLG entry**

Adds device-specific operation flags for the D-07X:

- `QUIRK_FLAG_IFACE_DELAY`: Wait during interface initialization
- `QUIRK_FLAG_FORCE_IFACE_RESET`: Force interface reset
- `QUIRK_FLAG_IGNORE_CLOCK_SOURCE`: Skip clock source validation

### sound/usb/endpoint.c (1 change · Raspberry Pi 4B only)

Adds D-07X-specific processing to the feedback endpoint handling to address an issue specific to the VL805 USB controller used in Raspberry Pi 4B.

### sound/usb/clock.c (1 change)

Resolves a clock validity check timeout that occurs during track changes by applying the `QUIRK_FLAG_IGNORE_CLOCK_SOURCE` flag to the clock validity check as well.

---

## Installation

### Prerequisites
- moOde Audio Player installed and running
- SSH access available
- Internet connection available

### 1. Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

After completion, reboot:

```bash
sudo reboot
```

### 2. Verify kernel version

After reconnecting:

```bash
uname -r
```

Expected output:
- Raspberry Pi 4B: `6.12.75+rpt-rpi-v8`
- Raspberry Pi 5B: `6.12.75+rpt-rpi-2712`

### 3. Install build tools and git

```bash
sudo apt install -y git build-essential bc bison flex libssl-dev libelf-dev
```

### 4. Fetch kernel source

```bash
cd ~
git clone --depth 1 --branch rpi-6.12.y --no-checkout https://github.com/raspberrypi/linux.git rpi-linux
cd rpi-linux
git sparse-checkout init
git sparse-checkout set sound/usb
git checkout
```

### 5. Modify quirks.c

Use Python interactive mode:

```bash
python3
```

Execute the following lines one at a time:

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

If `True` is displayed, continue:

```python
content = content.replace(old1, new1)
```

```python
old2 = 'DEVICE_FLG(0x0644, 0x8044, /* Esoteric D-05X */'
new2 = 'DEVICE_FLG(0x0644, 0x802c, /* Esoteric D-07X */\n\t   QUIRK_FLAG_IFACE_DELAY | QUIRK_FLAG_FORCE_IFACE_RESET |\n\t   QUIRK_FLAG_IGNORE_CLOCK_SOURCE),\n' + old2
print(old2 in content)
```

If `True` is displayed, continue:

```python
content = content.replace(old2, new2)
f = open('sound/usb/quirks.c', 'w')
f.write(content)
f.close()
exit()
```

Verify the changes:

```bash
grep -n "D-07X" sound/usb/quirks.c
```

The following 2 lines should appear:
1953:     ep->chip->usb_id == USB_ID(0x0644, 0x802c) ||  /* Esoteric D-07X /
2220: DEVICE_FLG(0x0644, 0x802c, / Esoteric D-07X */

### 6. Modify clock.c

```bash
nano +314 ~/rpi-linux/sound/usb/clock.c
```

Change this:
```c
if (validate && !uac_clock_source_is_valid(chip, fmt,
    entity_id)) {
```

To this:
```c
if (validate && !(chip->quirk_flags & QUIRK_FLAG_IGNORE_CLOCK_SOURCE) &&
    !uac_clock_source_is_valid(chip, fmt,
    entity_id)) {
```

Save with `Ctrl+O` → Enter → `Ctrl+X`.

### 7. Modify endpoint.c (Raspberry Pi 4B only)

> **Note: This step is not required for Raspberry Pi 5B**

```bash
nano +1878 ~/rpi-linux/sound/usb/endpoint.c
```

Change this:
```c
if (unlikely(sender->tenor_fb_quirk)) {
```

To this:
```c
if (unlikely(sender->tenor_fb_quirk ||
    sender->chip->usb_id == 0x0644802c)) {
```

Save with `Ctrl+O` → Enter → `Ctrl+X`.

### 8. Build

**For Raspberry Pi 4B:**

```bash
cd ~/rpi-linux
make -C /usr/src/linux-headers-6.12.75+rpt-rpi-v8 M=$(pwd)/sound/usb modules 2>&1 | tail -10
```

**For Raspberry Pi 5B:**

```bash
cd ~/rpi-linux
make -C /usr/src/linux-headers-6.12.75+rpt-rpi-2712 M=$(pwd)/sound/usb modules 2>&1 | tail -10
```

If the last line shows the following, the build was successful:
make: Leaving directory '/usr/src/linux-headers-6.12.75+rpt-rpi-xxxx'

### 9. Install

> **Important:** Raspberry Pi OS prioritizes loading compressed modules (`.ko.xz`). You must delete this file before installing, otherwise the fix will not take effect.

**For both Raspberry Pi 4B and 5B:**

```bash
sudo rm /lib/modules/$(uname -r)/kernel/sound/usb/snd-usb-audio.ko.xz
sudo cp sound/usb/snd-usb-audio.ko /lib/modules/$(uname -r)/kernel/sound/usb/
sudo depmod -a
sudo reboot
```

---

## moOde Configuration (After completing step 9)

### 10. Configure D-07X USB mode

Change the USB input setting on the D-07X to **HS_2** using the MENU button:

1. Press MENU repeatedly until `USB>***` is displayed
2. Use the VOLUME (+/-) button to select `HS_2`
3. Press the INPUT button or wait 10 seconds to exit the settings mode

### 11. Connect D-07X via USB

Connect the D-07X to the Raspberry Pi using a USB cable. Either USB 2.0 or USB 3.0 ports will work.

### 12. Configure moOde audio device

1. Open a browser and go to `http://moode.local` or the IP address
2. Click the「m」icon in the top right → **Configure** → **Audio**
3. Select「ESOTERIC USB AUDIO DEVICE」as the **Output device**
4. Set **Volume type** to「Fixed」
5. Click **Save**

### Additional configuration for OpenHome

1. Click the「m」icon → **Configure** → **Audio** → **Renderers**
2. Enable the **UPnP Client for MPD** service
3. Click **EDIT** → **General** → Change **Service type** to「OpenHome」
4. Click **Save**

---

## Verification

Check that the D-07X is recognized:

```bash
aplay -l | grep ESOTERIC
```

The following output indicates success:
card X: DEVICE [ESOTERIC USB AUDIO DEVICE], device 0: USB Audio [USB Audio]

To check transfer frequency during playback (example at 96kHz):

```bash
cat /proc/asound/DEVICE/stream0
```

If `Momentary freq` shows a stable value close to the playback frequency, the fix is working correctly.

### Disable USB auto-suspend

Linux may suspend USB devices after a period of inactivity, which can cause unexpected playback interruptions. To prevent this:

```bash
sudo nano /etc/udev/rules.d/99-usb-autosuspend.rules
```

Enter the following and save:
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="0644", ATTR{idProduct}=="802c", ATTR{power/control}="on"

Save with `Ctrl+O` → Enter → `Ctrl+X`, then apply the settings:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## Test Results

24-hour continuous playback tests were conducted at all sample rates in the author's environment, confirming stable operation.

※ Transfer frequency values show the minimum/average/maximum observed during testing. Values slightly above the target are normal behavior for ASYNC mode.

### Raspberry Pi 4B

| Sample Rate | Min | Average | Max | Result |
|-------------|-----|---------|-----|--------|
| 44.1kHz | 44104Hz | 44106.3Hz | 44605Hz | ✅ Pass |
| 48kHz | 48004Hz | 48006.6Hz | 48506Hz | ✅ Pass |
| 88.2kHz | 88209Hz | 88211.7Hz | 88711Hz | ✅ Pass |
| 96kHz | 96010Hz | 96013.2Hz | 96512Hz | ✅ Pass |
| 176.4kHz | 175922Hz | 176423.9Hz | 176924Hz | ✅ Pass |
| 192kHz | 192023Hz | 192025.6Hz | 192525Hz | ✅ Pass |

### Raspberry Pi 5B

| Sample Rate | Min | Average | Max | Result |
|-------------|-----|---------|-----|--------|
| 44.1kHz | 44100Hz | 44103.0Hz | 44602Hz | ✅ Pass |
| 48kHz | 48000Hz | 48002.6Hz | 48502Hz | ✅ Pass |
| 88.2kHz | 88201Hz | 88204.7Hz | 88705Hz | ✅ Pass |
| 96kHz | 96002Hz | 96005.2Hz | 96504Hz | ✅ Pass |
| 176.4kHz | 176406Hz | 176409.5Hz | 176910Hz | ✅ Pass |
| 192kHz | 192008Hz | 192010.9Hz | 192510Hz | ✅ Pass |

### Notes

Transfer frequencies slightly above the target are normal ASYNC mode behavior. The D-07X requests slightly more data than needed to prevent buffer underruns. The Raspberry Pi 5B shows transfer frequencies closer to the target values, indicating more precise USB transfer control.

---

## Important Notes

### Rebuilding after kernel updates

If the Raspberry Pi OS kernel is updated, the installed module will become invalid and the fix will no longer be effective. You will need to either stop automatic kernel updates or repeat steps 4-9 to rebuild after each update.

To check if the kernel version has changed:

```bash
uname -r
```

### Deleting the .xz compressed file

Raspberry Pi OS prioritizes loading the compressed version of kernel modules (`.ko.xz`). Always delete this file before installing. The same deletion is required when rebuilding after a kernel update.

### D-07X device settings

If the USB input setting is anything other than **HS_2** (NORM or HS_1), this fix will have no effect. Make sure to set it to HS_2.

### Support

This repository is based on personal testing. No warranty is provided and no support can be offered. Use at your own risk.

---

## Credits

The research, implementation, and verification of this fix was carried out with the support of Anthropic's AI assistant "Claude". Those facing similar challenges may find that consulting Claude can help guide them toward a solution.
