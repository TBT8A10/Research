# ENACOM TBT8A10 Tablet Research
**Disclaimer**: This document is not a guide of any kind. It's just a collection of unsorted personal notes I made while researching this tablet. \
I'm **NOT RESPONSIBLE** for any damage that may be caused to your device.

### Special Modes
* **MASKROM/BootROM Mode**: Allows us to flash the firmware.
* **LOADER/Rockusb  Mode**: Allows us to flash the firmware, flash ANY partition and dump the firmware.
* **Uboot**: One of the device's bootloaders. Handles pretty much the whole boot process.
    * **fastboot**: Useful to disable AVB and flash boot/recovery partitions. Can't flash super. The only way to access this mode is with `adb reboot-bootloader` or through the "Reboot to bootloader" option inside the recovery.
* **Recovery**:
    * **fastbootd**: Useful to check whether Android Verified Boot (AVB) is unlocked and to flash boot/recovery/super (system, vendor, odm, product) partitions

### Key combinations
With the tablet OFF:
* **Enter Loader Mode**: Hold volume Up/Down and connect USB cable
* **Enter Recovery**: Hold Volume Up/Down + Power

With the tablet ON:
* **Reboot**: Hold either Volume down + Power or just the power button
    * An alternative is to use a toothpick to press the button under the power button.
* **Shutdown**: Volume up + Power

### Boot Sequence
[Wiki Page](https://opensource.rock-chips.com/wiki_Rockusb)|[Wiki Page](https://opensource.rock-chips.com/wiki_Boot_option) \
It's possible that the device can boot from an SD Card in [multiple stages](http://rockchip.wikidot.com/boot-sequence)
1. BootROM (Hard-coded, can't be modified)
   * If no bootable firmware found, enters MASKROM mode.
   * According to Rockchip, it's possible to "short the eMMC clock to GND" to force the device into MASKROM.
2. miniloader/idbloader (MiniLoaderAll.bin in firmware)
   * "Usbplug" Mode (similar to MASKROM?) (rk3368_usbplug_v2.6 binary inside MiniLoaderAll.bin)
   * Rockusb mode (LOADER mode) (NOT CONFIRMED)
3. From experience, if something went wrong at this point (e.g. uboot is corrupted and can't be loaded) device will enter MASKROM mode.
4. Uboot
    * If the respective key combination are pressed, enters Uboot's LOADER mode or Android's recovery.
    * If SD Card is detected and has a bootable firmware flashed, boots it.
        * I couldn't manage to actually boot something from an SD Card other than Uboot (and thus LOADER mode) which is enough to unbrick the device in some cases (more info below).
    * If the boot image is fine, boots it.
    * If there's a bootable USB device connected, boots it (Untested)
5. Linux Kernel (boot or recovery)
    * Displays the ENACOM logo

### Tools
#### Rockchip Tools
Use the following versions. Newer or older ones may not work properly.
* **DriverAssitant_v4.5**: Drivers required to detect the device while on LOADER/MASKROM mode.
* **AndroidTool_Release_v2.71**: Can flash the firmware (LOADER/MASKROM mode), flash partitions (LOADER mode) and dump firmware (LOADER mode).
* **FactoryTool_v1.45_bnd**: Can flash the firmware. I've never needed to use it.
* **SD_Firmware_Tool._v1.46**: Can flash the firmware to a SD Card.
#### Other Tools
* [imgRePackerRK](https://forum.xda-developers.com/t/tool-imgrepackerrk-rockchips-firmware-images-unpacker-packer.2257331/): Can be used to unpack/repack Rockchip firmware and partition images.
* [rkDumper](https://forum.xda-developers.com/t/tool-rkdumper-utility-for-backup-firmware-of-rockchips-devices.2915363/): Can be used to dump the firmware through LOADER mode (only the first 32 MB of the EMMC, more info below)
* [adbDumper](https://forum.xda-developers.com/t/tool-adbdumper-utility-for-backup-firmware-of-android-devices.4525721/): Once root is acquired, can be used to dump the firmware through ADB.

### Firmware
Official firmware is available on [Google Drive](https://drive.google.com/file/d/12YQDCDvujEDlx5ZTQb0ChDukXJs9QSd6/view?usp=drive_link). \
If you receive OTA updates, it's not suggested to update as a lot of users have bricked the tablets by doing so (probably a soft-brick). \
Device is unlockable up to official 08/05/2023 build. No idea about newer builds.

### Dumping firmware
~~This is not possible in our device~~. Update: May be possible, more info below. \
Only the first 32 MB of the EMMC can be read through LOADER Mode, either through AndroidTool or RedScorpio's rkdumper tool.
Thus, only the following partitions can be dumped:
* uboot
* trust
* misc
* dtb
* dtbo
* vbmeta
* boot (just a few bytes)

~~The only way to dump firmware is to unlock bootloader, obtain root, and use adbDumper.~~. \
**Update:** Dumping the whole firmware through LOADER mode may be possible [doing this](https://forum.xda-developers.com/t/tool-rkdumper-utility-for-backup-firmware-of-rockchips-devices.2915363/post-88697849).

### Unlock bootloader
I'm 90% sure that the bootloader comes unlocked out of the box, but we can't flash boot and partitions inside the super partition unless we disable Android Verified Boot (AVB).
1) Enable OEM Unlock in Android developer settings (probably not required, but lets do it just in case)
2) Reboot into uboot's fastboot (`adb reboot-bootloader`)
3) `fastboot oem at-unlock-vboot` (this will unlock verified boot, allowing to flash any recovery/boot/super (system, vendor, odm, product)
4) `fastboot flashing unlock` (Probably irrelevant) (unlocks flashing boot/recovery and maybe other partitions through uboot's fastboot)
5) `fastboot flashing unlock_critical` (Probably irrelevant) (not sure what it unlocks or if it does anything)
6) Now recovery's fastbootd should report device's unlocked when executing `fastboot getvar unlocked`

To re-enable AVB, `fastboot oem at-lock-vboot` may do the trick (untested).

### Unbrick
**Soft-brick**: Uboot works and we can get into LOADER mode
1. Boot into LOADER mode.
2. Open Rockchip's AndroidTool
3. Two options:
    * Load our device's partition table (or create it manually by pressing "Dev partition" & analyzing its output) & flash the corrupted partition
    * Reflash stock firmware (Upgrade firmware tab)

**Hard-brick**: Can't get into LOADER mode
* Device boots into **MASKROM Mode**:
    * Flash stock firmware with AndroidTool
* Device doesn't show any signs of life:
    * Supposedly we can force MASKROM Mode by shorting the eMMC (I don't know how to do this)
    * It's possible that Uboot still works even if LOADER mode doesn't. You can check this doing the following:
        1. Connect SD Card to the computer & flash stock firmware to it using Rockchip's SD_Firmware_Tool
        3. Insert Card into Tablet
        4. Reboot tablet. It should now show ENACOM logo.
        5. Try to boot into LOADER Mode. It will either boot into MASKROM or LOADER, it doesn't matter.
        6. Flash stock firmware normally
        7. Tablet will reboot and show ENACOM Logo. Disconnect SD Card and reboot.
        8. Tablet will boot into recovery, perform factory reset (wipe /data, /cache, etc.) and reboot into system.

### Hardware Info
* **WiFi & Bluetooth Chip**: Unisoc uwe5623 (similar to uwe5621)
    * Firmware: /vendor/etc/firmware/wcnmodem.bin
    * Driver version: W21.10.3
* **GPU**: PowerVR Rogue G6110 (BVNC 5.9.1.46)
    * Blobs & their version
        * /vendor/ Drivers:
            * bin
                * pvrsrvctl | UM P 5.38
                * pvrtld | UM Q 5.58
            * firmware
                * rgx.fw.signed.5.9.1.46 | UM Q 5.56
            * lib
                * egl
                    * egl.cfg
                    * libEGL_POWERVR_ROGUE.so | UM Q 5.56
                    * libGLESv1_CM_POWERVR_ROGUE.so | UM Q 5.56
                    * libGLESv2_POWERVR_ROGUE.so | UM Q 5.58
                * hw
                    * gralloc.rk3368.so | UM Q 5.58
                    * memtrack.rk3368.so | UM Q 5.56
                    * vulkan.rk3368.so | Not implemented
                * modules
                    * pvrsrvkm.ko | UM Q 5.58
                * libGLESv2_POWERVR_ROGUE.so | Not implemented
                * libIMGegl.so | UM Q 5.58
                * libLLVMIMG.so | 11/11/2016
                * libPVROCL.so | UM Q 5.56
                * libPVRScopeServices.so | UM Q 5.58
                * libclangIMG.so | 11/11/2016
                * libcreatesurface.so | UM N5.10
                * libglslcompiler.so | UM Q 5.56
                * liboclcompiler.so | UM N5.10
                * libpvrANDROID_WSEGL.so | UM Q 5.56
                * libsrv_init.so | Not implemented
                * libsrv_um.so | UM Q 5.58
                * libufwriter.so | UM Q 5.56
                * libusc.so | UM Q 5.56
            * lib64
                * Not implemented
            * etc
                * powervr.ini
