## Android

### Prerequisites

* Enable "Developer Options" by going into "Settings", "About this phone", and then tap on the "Build Number" field seven times
* Enable "USB Debugging" in "Developer Options" on the phone
* Enable "Allow OEM Unlock" in "Developer Options" on the phone
* Download [Android Platform Tools](https://developer.android.com/studio/releases/platform-tools.html#download)
* If on Windows, download the [Motorola USB Driver for Windows](http://www.teamandroid.com/2015/06/24/moto-e-2015-usb-drivers-download)
  * Also called Motorola Device Manager

### Backing up Your Phone

* To backup your phone, run "adb backup -apk -shared -all -system"
* To restore your phone, run "adb restore backup.ab"

References:
* [GUIDE: Full Phone Backup without Unlock or Root](https://forum.xda-developers.com/galaxy-nexus/general/guide-phone-backup-unlock-root-t1420351)

### Unlocking the Moto E2 LTE XT1527 (surnia)

* Run "adb reboot bootloader"
* When the phone finishes entering fastboot, run "fastboot devices" to verify that it's properly connect
* Run "fastboot oem get_unlock_data"
* Go to the [Motorola Support Device Unlocking](http://motorola-global-portal.custhelp.com/app/standalone/bootloader/unlock-your-device-a) page, and enter the unlock data, then click the button to request an unlock code.
* Run "fastboot oem unlock <UNIQUE_KEY>", using the unique key e-mailed to you from Motorola Support

References:
* [Install LineageOS on surnia](https://wiki.lineageos.org/devices/surnia/install)
* [Android Platform Tools](https://developer.android.com/studio/releases/platform-tools.html#download) - easy way to get adb without downloading the whole SDK
* [Motorola USB Driver for Windows](http://www.teamandroid.com/2015/06/24/moto-e-2015-usb-drivers-download)
* [Motorola Support Device Unlocking](http://motorola-global-portal.custhelp.com/app/standalone/bootloader/unlock-your-device-a)

### Install TWRP Custom Recovery

* Now that the phone has completely re-initialized, re-enable "USB Debugging", as before
* Download the Moto E LTE (surnia) twrp image from the [twrp website](http://twrp.me/)
* Run "adb reboot bootloader"
* Run "fastboot devices" to make sure it's still being discovered properly
* Run "fastboot flash recovery twrp-x.x.x-x-surnia.img"
* Reboot into recovery to verify installation

References:
* [Install LineageOS on surnia](https://wiki.lineageos.org/devices/surnia/install)

### Install ROM

* Download the ROM image for installation, e.g. [LineageOS](https://download.lineageos.org/surnia), [Cyanogenmod](https://forum.xda-developers.com/moto-e-2015/orig-development/rom-cyanogenmod-13-moto-e-lte-t3329100).
* Download the [Google Apps package](http://opengapps.org/?api=7.1&variant=micro), if desired
* Push each downloaded zip to the sdcard: "adb push filename.zip /sdcard/"
* In recovery mode, select "Wipe", then "Advanced Wipe"
  * Select "Cache", "System", and "Data" partitions to be wiped
* Back on the main menu of recovery mode, select "Install"
* Select the ROM zip
* Follow the prompts to install it
  * DO NOT REBOOT after installing ROM if you intend to install Gapps
* Repeat for any additional zips pushed to the sdcard
* Reboot into System - I opted not to install twrp app

References:
* [Install LineageOS on surnia](https://wiki.lineageos.org/devices/surnia/install)
* [Install Cyanogenmod on surnia](https://forum.xda-developers.com/moto-e-2015/orig-development/rom-cyanogenmod-13-moto-e-lte-t3329100)

### Modems and Basebands

* Extract the zip file and, while booted into fastboot, run "2.flash.bat"

References:
* [Download firmware](https://forum.xda-developers.com/moto-e-2015/general/moto-e-2015-modems-t3296044)

### Kernels

* Recent Squid kernels (r19+) require Android 6.0+ ROMs
  * May require a modem update (r12+), too
* Use adb to push the zip to the sdcard
* Use twrp to install it

References:
* [Squid Kernel for Moto E LTE (2015)](https://forum.xda-developers.com/moto-e-2015/orig-development/squid-kernel-moto-e-lte-2015-t3072209)

### Remove Bootloader Unlocked Warning

* Download [alternate boot logo](http://www.mediafire.com/download/1x0s6gbxmrw5x0r/Moto-E-2015-logo.zip)
* Extract it into folder with fastboot.exe
* Reboot into bootloader with "adb reboot bootloader"
* Flash the new logo "fastboot flash logo logo.bin"
* Reboot "fastboot reboot"

References:
* [Remove bootloader unlocked warning](http://www.droidviews.com/how-to-remove-bootloader-unlocked-warning-on-moto-e-2015/)
