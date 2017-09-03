## Lede/OpenWRT

### System Setup
* If the system is EFI capable, ensure it's configured to boot in EFI mode (not BIOS/CSM/legacy mode)
* Disable any BIOS features that aren't needed in a router
  * Network/PXE boot
  * Audio hardware
  * Serial hardware (although this can actually be useful for debugging)
* Set BIOS clock to UTC
* If there's a setting for what to do after power loss, set it to always power on
* Disable secure boot
* Set default boot device to be the EFI Shell App (it will run the startup.nsh script to boot the kernel with the correct parameters)

For reasons I don't completely understand, the kernel couldn't find the root partition when my hard drive was attached to anything other than the primary sata port. Whether EFI or CSM (BIOS), specifying the root partition by sdx# or partition UUID, nothing worked. Moving the hard drive to sata port 0 made everything magically start working.

### Quick Guide to Legacy BIOS Boot Install
* [OpenWRT on x86-64](https://we.riseup.net/lackof/openwrt-on-x86-64)


* Download a [LEDE release](https://downloads.lede-project.org/releases/) combined ext4 image
  * `...-combined-squashfs.img` for wear-intolerant storage like USB, SD card, etc
  * `...-combined-ext4.img` for high-wear storage like SSDs
* Gunzip the image, and copy the resulting file into `/mnt/<usbkey>/data/lede` (if using SystemRescueCD)
* Boot the router with the USB key
* Write the selected image file to the router's hard drive: `cat /livemnt/boot/data/lede/lede-...-x86-64-combined-ext4.img > /dev/sda
* Resize the root partition to fill the disk:
  * Make note of the start sector of partition 2: `parted /dev/sda unit s p`
  * Delete the second partition: `parted /dev/sda rm 2`
  * Make the new partition, using the same starting sector: `parted /dev/sda unit s mkpart primary ext4 <start sector> 100%`
    * If prompted about improper alignment, Ignore
  * Resize the filesystem to fill the partition: `resize2fs /dev/sda2`
  * `reboot`

### Full Guide to EFI Install
#### Preparing the bootable USB key
* [LEDE releases](https://downloads.lede-project.org/releases/)

Options:
* [Ubuntu](https://wiki.ubuntu.com/LiveUsbPendrivePersistent)
* [SystemRescueCD](http://www.system-rescue-cd.org/Installing-SystemRescueCd-on-a-USB-stick/)
* [Kali](https://docs.kali.org/downloading/kali-linux-live-usb-install)
* [Unetbootin](https://unetbootin.github.io/)

I used SystemRescueCD for my bootable linux USB key.

* Put a bootable linux OS on a USB key
* Mount the USB key
* Download a [LEDE release](https://downloads.lede-project.org/releases/) rootfs ext4 image and vmlinuz
  * pick a release: `targets/x86/64/`
  * see [this page](https://we.riseup.net/lackof/openwrt-on-x86-64#using-the-combined-images) for more on the subject of combined images
* Gunzip the image, and copy the resulting file into `/mnt/<usbkey>/data/lede-efi` (if using SystemRescueCD)
* Copy the vmlinuz into `/mnt/<usbkey>/data/lede-efi`

#### Installation
* [Guide to installing OpenWRT on MinnowBoards](http://elinux.org/Minnowboard:MinnowMaxDistros#OpenWrt)
* [ArchLinux UEFI boot guide](https://wiki.archlinux.org/index.php/GNU_Parted#UEFI.2FGPT_examples)
* [OpenWRT documentation for x86](https://wiki.openwrt.org/inbox/doc/openwrt_x86)

* On router computer, boot from the USB key
* Change directories to where you put the image: `cd /livemnt/boot/data`
* Convert from MBR to GPT
  * using parted: `parted /dev/sda mklabel gpt`
  * using gdisk: `gdisk /dev/sda`, then write the changes and exit
* Using cgdisk, or some other gpt-enabled partitioning tool:
  * create an ESI partition (`ef00`) of between 64 and 256MB, and label it BOOT
  * create a linux partition (`8300`) using the rest of the disk, and label it ROOT
* Format the partitions:
  * `mkfs.vfat -F 32 -n BOOT /dev/sda1` (`-F 32` => FAT32, `-n BOOT` => set volume name to `BOOT`)
  * `mkfs.ext4 -L ROOT /dev/sda2` (`-L ROOT` => set volume name to `ROOT`)
* Set up the EFI boot partition:
  * Mount to `BOOT` partition: `mount /dev/sda` /mnt/bootfs`
  * Create the EFI boot image directory: `mkdir -p /mnt/bootfs/EFI/BOOT`
  * Copy the kernel into the boot image directory: `cp /livemnt/boot/data/lede-efi/lede-...-x86-64-vmlinuz /mnt/bootfs/EFI/BOOT/bootx64.efi`
  * Create the EFI startup script to pass the necessary kernel params: `echo 'fs:\\EFI\\BOOT\\bootx64.efi root=-/dev/sda2 rootfstype=ext4 rootwait console=tty0 noinitrd" > /mnt/bootfs/startup.nsh`
* Set up the root partition:
  * Mount the `ROOT` partition: `mount /dev/sda2 /mnt/rootfs`
  * Mount the rootfs image: `mount -o loop /livemnt/boot/data/lede-...-x86-64-rootfs-ext4.img /mnt/ledefs`
  * Copy all the files over: `rsync -avxHAWX --numeric-ids --info=progress2 /mnt/ledefs/ /mnt/rootfs/`
* `reboot`
* **NOTHING WILL DISPLAY WHEN THE SYSTEM STARTS BOOTING.  DON'T PANIC: LEDE 17.0x KERNELS DON'T HAVE EFI FRAMEBUFFER CONFIGURED.**


### Configuration
* [LEDE Quick Start guide](https://lede-project.org/docs/guide-quick-start/start)

* From a system connected to the LAN port of the router, ssh into `192.168.1.1`
* Set a password: `passwd`
* If the router is being setup with a private network connect to the WAN port, specifically if the upstream network is `192.168.1.x`:
  * change the router's IP to a different subnet: `uci set network.lan.ipaddr=192.168.2.1 && uci commit network && reload_config`
  * disconnect and reconnect the _client_ computer to the router to get an ip on the new network (either physically, or run `sudo ifdown eth0 && sudo ifup eth0`)
  * In the rest of the guide, substitute `192.168.2.x` for references to `192.168.1.x`
* Install the `luci` web interface: `opkg update; opkg install luci`
* Open a web browser, and type `http://lede/` into the navigation bar
* System => System
  * Set a `Hostname`
  * Set the `Timezone`
* Network => DHCP and DNS
  * Set a `Local server`
  * Set a `Local domain` (should generally have the same value as `Local server`)


### Bonus Round
* [LuCi https certificate](https://lede-project.org/docs/user-guide/getting-rid-of-luci-https-certificate-warnings)
* [Smart Queue Management (SQM)](https://lede-project.org/docs/user-guide/sqm) - minimizing [bufferbloat](https://www.bufferbloat.net/projects/bloat/wiki/What_can_I_do_about_Bufferbloat/)
* luci-app-adblock
* luci-app-ddns

#### Cisco/Fortigate/etc IPSEC VPN
* System => Software
  * Download and install package: vpnc
  * ssh into the router, and edit `/etc/vpnc/default.conf`
  * Download vpnc-script from [here](http://git.infradead.org/users/dwmw2/vpnc-scripts.git/blob_plain/HEAD:/vpnc-script) and save it as /etc/vpnc/vpnc-script (and be sure to set +x on it)

#### OpenConnect SSL VPN
* https://wiki.openwrt.org/doc/howto/openconnect-setup
