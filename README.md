# ENACOM TBT8A10 Tablet Research
**Disclaimer**: This document is not a guide of any kind. It's just a collection of unsorted personal notes I made while researching this tablet. \
I'm **NOT RESPONSIBLE** for any damage that may be caused to your device.

### Hardware Info
The tablet was built by [Incartech](http://www.incartech.com.cn/). \
Information about the hardware and blobs/drivers can be found on the [vendor repo](https://github.com/TBT8A10/android_vendor_incar_tbt8a10/tree/main).

### Special Modes
* **MASKROM/BootROM Mode**: Allows us to flash the firmware.
* **Uboot**: One of the bootloaders. Handles pretty much the whole boot process.
    * **LOADER/Rockusb  Mode**: Allows us to flash partitions, firmware and dump it.
    * **fastboot**: Useful to disable AVB and flash boot/recovery partitions. Can't flash super. The only way to access this mode is with `adb reboot-bootloader` or through the "Reboot to bootloader" option inside recovery.
* **Recovery**:
    * **fastbootd**: Useful to check whether Android Verified Boot (AVB) is unlocked and to flash boot/recovery/super (system, vendor, odm, product) partitions

### Key combinations
With the tablet OFF:
* **Enter Loader Mode**: Hold volume Up/Down and connect USB cable
* **Enter Recovery**: Hold Volume Up/Down + Power

With the tablet ON:
* **Reboot**: Hold either Volume down + Power or just the power button
    * An alternative is to use a toothpick to press the button under the power button.
* **Shutdown**: Hold Volume up + Power

### Boot Sequence
[Wiki Page](https://opensource.rock-chips.com/wiki_Rockusb)|[Wiki Page](https://opensource.rock-chips.com/wiki_Boot_option) \
It's possible that the device can boot from an SD Card in [multiple stages](http://rockchip.wikidot.com/boot-sequence)
1. **BootROM** (Hard-coded, can't be modified)
   * If no bootable firmware found, enters MASKROM mode.
   * According to Rockchip, it's possible to "short the eMMC clock to GND" to force the device into MASKROM.
2. **miniloader**/idbloader (MiniLoaderAll.bin in firmware)
   * "Usbplug" Mode (similar to MASKROM?) (rk3368_usbplug_v2.6 binary inside MiniLoaderAll.bin)
3. From experience, if something went wrong at this point (e.g. uboot is corrupted and can't be loaded) device will enter MASKROM mode.
4. **Uboot**
    * If the respective key combination is pressed, enters Uboot's LOADER mode or Android's recovery.
    * Finds a bootable boot image in the following order:
        * SD Card
            * I couldn't manage to boot anything from an SD Card other than Uboot (and thus LOADER mode) which is enough to unbrick the device in some cases
        * eMMC
        * USB Device (untested)
    * Loads resources from boot image (DTB, logo...)
    * Initializes some hardware like the display
    * Displays ENACOM logo
    * Boots the image
5. **Linux Kernel** (boot or recovery)

### Tools
#### Rockchip Tools
Use the following versions unless specified otherwise. Newer or older ones may not work properly.
* **DriverAssitant_v4.5**: Drivers required to detect the device while on LOADER/MASKROM mode.
* **AndroidTool_Release_v2.71**: Flash firmware (LOADER/MASKROM mode), flash partitions (LOADER mode) and dump firmware (LOADER mode).
* **FactoryTool_v1.45_bnd**: Flash firmware. I've never needed to use it.
* **SD_Firmware_Tool._v1.46**: Flash firmware to an SD Card.
#### Other Tools
* [imgRePackerRK](https://forum.xda-developers.com/t/tool-imgrepackerrk-rockchips-firmware-images-unpacker-packer.2257331/): Unpack/repack Rockchip firmware and partition images.
* [rkDumper](https://forum.xda-developers.com/t/tool-rkdumper-utility-for-backup-firmware-of-rockchips-devices.2915363/): Dump firmware through LOADER mode
* [adbDumper](https://forum.xda-developers.com/t/tool-adbdumper-utility-for-backup-firmware-of-android-devices.4525721/): Dump firmware through ADB (root required)

### Firmware
Official firmware is available on [Google Drive](https://drive.google.com/file/d/12YQDCDvujEDlx5ZTQb0ChDukXJs9QSd6/view?usp=drive_link). \
If you receive OTA updates, it's not suggested to update as a lot of users have bricked their tablets by doing so (probably a soft-brick).

Device is unlockable up to official 08/05/2023 build. No idea about newer builds.

### Dump firmware
Out of the box, only the first 32 MB of the eMMC can be read through LOADER Mode using either AndroidTool or rkdumper.
Additionally, LOADER mode can't access anything before uboot, so we can't dump the miniloader. \
Thus, only the following partitions can be dumped:
* uboot (start=16384 count=8192)
* trust (start=24576 count=8192)
* misc (start=32768 count=8192)
* dtb (start=40960 count=8192)
* dtbo (start=49152 count=8192)
* vbmeta (start=57344 count=2048)

However, we can patch uboot to allow us to dump up to 4 GB of the eMMC:
1. Grab uboot.img from firmware or dump uboot from device
2. Unpack uboot.img with imgRePackerRK
3. Open uboot.bin with an hex editor and search for "81 00 01 8B 24 00 02 8B 9F 40 40 F1 49 01 00 54"
4. Replace "40 40" with "00 60" and save it
5. Repack uboot.img & flash it

To dump, use **AndroidTool_Release_v2.38** as newer versions don't allow to dump after 32 MB.
Now we can dump these partitions in addition to the previous ones:
* boot (start=59392 count=65536)
* security (start=124928 count=8192)
* recovery (start=133120 count=196608)
* backup (start=329728 count=229376)
* cache (start=559104 count=786432)
* metadata (start=1345536 count=32768)
* frp (start=1378304 count=1024)
* super (start=1379328 count=6373376)

### Unlock bootloader
I'm 90% sure that the bootloader comes unlocked out of the box, but we can't flash boot and partitions inside the super partition unless we disable Android Verified Boot (AVB).
1) Enable OEM Unlock in Android developer settings (probably not required, but do it just in case)
2) Reboot into uboot's fastboot (`adb reboot-bootloader`)
3) `fastboot oem at-unlock-vboot` (this will unlock verified boot, allowing to flash any recovery/boot/super (system, vendor, odm, product)
4) `fastboot flashing unlock` (Probably irrelevant) (unlocks flashing boot/recovery and maybe other partitions through uboot's fastboot)
5) `fastboot flashing unlock_critical` (Probably irrelevant) (not sure what it unlocks or if it does anything)
6) Now recovery's fastbootd should report device's unlocked when executing `fastboot getvar unlocked`

To re-enable AVB, `fastboot oem at-lock-vboot` may do the trick (untested).

### Unbrick
**Soft-brick**: Uboot works and we can get into LOADER mode
1. Boot into LOADER mode.
2. Open AndroidTool
3. Two options:
    * Load our device's partition table (or create it manually by pressing "Dev partition" & analyzing its output) & flash the corrupted partition
    * Reflash stock firmware (Upgrade firmware tab)

**Hard-brick**: Can't get into LOADER mode
* Device boots into **MASKROM Mode**:
    * Flash stock firmware with AndroidTool
* Device doesn't show any signs of life:
    * Supposedly we can force MASKROM Mode by shorting the eMMC (I don't know how to do this)
    * It's possible that Uboot or miniloader still works, and may be able to boot from an SD Card.
        1. Connect SD Card to the computer & flash stock firmware to it using Rockchip's SD_Firmware_Tool
        3. Insert Card into Tablet
        4. Reboot tablet. It should now show ENACOM logo.
        5. Try to boot into LOADER Mode. It will either boot into MASKROM or LOADER, it doesn't matter.
        6. Flash stock firmware normally
        7. Tablet will reboot and show ENACOM Logo. Disconnect SD Card and reboot.
        8. Tablet will boot into recovery, perform factory reset (wipe /data, /cache, etc.) and reboot into system.

______
## Customizing the tablet
I asked Incar for the kernel and U-Boot's source code, but sadly they can't share it.

### Building U-Boot
Information available on the [U-Boot repo](https://github.com/TBT8A10/u-boot).

### Building Android Kernel
As of now, I wasn't able to boot a kernel built by me; it freezes at an early stage. This may not be feasible due to all the changes Incar has made to Rockchip's kernel.

### DTB & Enacom Logo
The boot image contains a DTB just like most Android devices. However, that one is not used. \
Additionally, it contains a "second" file (unpack with [AIK](https://forum.xda-developers.com/t/tool-android-image-kitchen-unpack-repack-kernel-ramdisk-win-android-linux-mac.2073775/) and it will be inside the `split_img` folder) which can be unpacked/repacked with imgRePackerRK. Inside it is the real DTB and the ENACOM splash screen bitmap.

I've extracted the DTS from the DTB and cleaned it using my [Python script](https://github.com/TBM13/dts_cleaner) to make it more readable. It's [available here](./Resources/stock_dts_cleaned.txt).

### Building TWRP
I managed to build TWRP using Rockchip's device tree (had to make a lot of modifications and quick hacks) and the stock prebuilt kernel. It loads enough to make ADB work but the recovery itself crashes due to some graphic-related errors.\
<details>
<summary>View log</summary>

```
Starting TWRP 3.7.0_11-0-063d211c on Sun Jan 29 23:41:48 2023
 (pid 151)
TW_NO_BATT_PERCENT := true
TW_NO_USB_STORAGE := true
PRODUCT_USE_DYNAMIC_PARTITIONS := true
I:Found brightness file at '/sys/devices/platform/backlight/backlight/backlight/brightness'
I:Got max brightness 255 from '/sys/devices/platform/backlight/backlight/backlight/max_brightness'
I:TWFunc::Set_Brightness: Setting brightness control to 255
I:TW_EXCLUDE_ENCRYPTED_BACKUPS := true
I:LANG: en
Starting the UI...
setting DRM_FORMAT_RGBX8888 and GGL_PIXEL_FORMAT_RGBX_8888
setting DRM_FORMAT_XBGR8888 and GGL_PIXEL_FORMAT_RGBA_8888
mmap() failed: Invalid argument
setting DRM_FORMAT_XBGR8888 and GGL_PIXEL_FORMAT_RGBA_8888
mmap() failed: Invalid argument
Forcing pixel format: RGB_565
failed to put force_rgb_565 fb0 info: Invalid argument
I:TWFunc::Set_Brightness: Setting brightness control to 255
TW_SCREEN_BLANK_ON_BOOT := true
I:Loading package: splash (/twres/splash.xml)
I:Load XML directly
I:PageManager::LoadFileToBuffer loading filename: '/twres/splash.xml' directly
I:Checking resolution...
libc: Fatal signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0 in tid 151 (recovery), pid 151 (recovery)
libc: Unable to set property "ro.twrp.boot" to "1": error code: 0xb
libc: Unable to set property "ro.twrp.version" to "3.7.0_11-0": error code: 0xb
```
</details>

If those issues can't be fixed, an alternative could be to disable the UI and control it through ADB.

Meanwhile, I made some modifications to the stock recovery to make it more comfortable. Will try to upload it soon.

### Building AOSP
This should be possible using Rockchip's device trees. However, the tablet supports Project Treble and GSIs work fine.

### Flashing a GSI
The kernel is built for arm64, but the vendor uses 32-bit binaries. Thus, only `arm32_binder64` variants of GSIs will work. I recommend to try [TrebleDroid](https://github.com/TrebleDroid/treble_experimentations/releases) or [phhusson's GSIs](https://github.com/phhusson/treble_experimentations) first.

#### Known issues on TrebleDroid/PHH and TrebleDroid/PHH-based GSIs:
* Boot animation doesn't work
* WiFi may not work on the first boot. If that's the case, try rebooting a few times and turning WiFi on/off
* Android 12 or higher don't boot due to issues with the GPU Driver. This can be fixed by updating them (more info soon).
* HDMI probably doesn't work (never tested it)
* Performance is terrible. The UI is very laggy and unresponsive. The stock ROM also suffers from this but it's less noticeable there.