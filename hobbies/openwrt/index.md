## Lede/OpenWRT

### Hardware

I built my own x86-64 router from the following:
* [m350 mini-itx case](http://www.mini-box.com/M350-universal-mini-itx-enclosure)
* [picoPSU-90 DC-DC power supply](http://www.mini-box.com/picoPSU-90)+[120W AC-DC power supply](http://www.mini-box.com/12v-10A-AC-DC-Power-Adapter)
* [SuperMicro X11SBA-F Pentium N3700 motherboard+cpu](http://www.supermicro.com/products/motherboard/X11/X11SBA-F.cfm)
    * Dual Gbit Intel ethernet adapters
    * Supports AES-NI instructions for hardware crypto acceleration
    * IPMI support enables network kvm access
* 8GB of RAM (way overkill, but the cost difference from 4GB was pretty minimal)
* 120GB Intel SATA SSD (again, overkill, but the cost difference was minimal)

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
    * See [my notes on building the openwrt kernel](/hobbies/openwrt/kernel.html) if you want to hardcode the kernel command line parameters and be able to boot the partition directly

For reasons I don't completely understand, the kernel couldn't find the root partition when my hard drive was attached to anything other than the primary sata port. Whether EFI or CSM (BIOS), specifying the root partition by sdx# or partition UUID, nothing worked. Moving the hard drive to sata port 0 made everything magically start working.

### Quick Guide to Legacy BIOS Boot Install

1. Download a [LEDE release](https://downloads.lede-project.org/releases/) combined ext4 image
    * `...-combined-squashfs.img` for wear-intolerant storage like USB, SD card, etc
    * `...-combined-ext4.img` for high-wear storage like SSDs
2. Gunzip the image, and copy the resulting file into `/mnt/<usbkey>/data/lede` (if using SystemRescueCD)
3. Boot the router with the USB key
4. Write the selected image file to the router's hard drive: `cat /livemnt/boot/data/lede/lede-...-x86-64-combined-ext4.img > /dev/sda
5. Resize the root partition to fill the disk:
    * Make note of the start sector of partition 2: `parted /dev/sda unit s p`
    * Delete the second partition: `parted /dev/sda rm 2`
    * Make the new partition, using the same starting sector: `parted /dev/sda unit s mkpart primary ext4 <start sector> 100%`
        * If prompted about improper alignment, Ignore
    * Resize the filesystem to fill the partition: `resize2fs /dev/sda2`
    * `reboot`

References:
* [OpenWRT on x86-64](https://we.riseup.net/lackof/openwrt-on-x86-64)

### Full Guide to EFI Install

#### Preparing the bootable USB key

Options:
* [Ubuntu](https://wiki.ubuntu.com/LiveUsbPendrivePersistent)
* [SystemRescueCD](http://www.system-rescue-cd.org/Installing-SystemRescueCd-on-a-USB-stick/)
* [Kali](https://docs.kali.org/downloading/kali-linux-live-usb-install)
* [Unetbootin](https://unetbootin.github.io/)

I used SystemRescueCD for my bootable linux USB key.

1. Put a bootable linux OS on a USB key
2. Mount the USB key
3. Download a [LEDE release](https://downloads.lede-project.org/releases/) rootfs ext4 image and vmlinuz
    * pick a release: `targets/x86/64/`
    * see [this page](https://we.riseup.net/lackof/openwrt-on-x86-64#using-the-combined-images) for more on the subject of combined images
4. Gunzip the image, and copy the resulting file into `/mnt/<usbkey>/data/lede-efi` (if using SystemRescueCD)
5. Copy the vmlinuz into `/mnt/<usbkey>/data/lede-efi`

References:
* [LEDE releases](https://downloads.lede-project.org/releases/)

##### systemd-boot (gummiboot)

1. Install systemd build deps: `sudo apt install meson gnu-efi gperf libcap-dev libmount-dev docbook-xsl`
2. Clone the systemd git repo: `git clone https://github.com/systemd/systemd.git && cd systemd`
3. Configure the build: `meson -D gnu-efi=true build/`
4. Build the efi bootloader: `ninja -C build src/boot/efi/systemd-bootx64.efi`
5. Copy the bootloader into a temporary EFI staging area: `install -D build/src/boot/efi/systemd-bootx64.efi systemd-boot/EFI/BOOT/bootx64.efi`
6. Copy the kernel into the EFI kernel staging area: `install -D /mnt/<usbkey>/data/lede-efi/lede-17.01.2-x86-64-vmlinuz systemd-boot/linux/`
7. Create the basic bootloader configuration: `install -d systemd-boot/loader && vi systemd-boot/loader/loader.conf`
  ```
  default lede-*
  timeout 3
  editor 0
  ```
8. Create the boot entry for the kernel we added: `install -d systemd-boot/loader/entries && vi systemd-boot/loader/entries/lede-17.01.2.conf`
  ```
  title    LEDE 17.01.2
  efi      /linux/lede-17.01.2-x86-64-vmlinuz
  options root=/dev/sda2 rootfstype=ext4 rootwait console=tty0 noinitrd
  ```
9. Copy the entire systemd-boot directory tree into`/mnt/<usbkey>/data/lede-efi`

References:
* [Arch systemd-boot documentation](https://wiki.archlinux.org/index.php/systemd-boot)
* [systemd-boot official website](https://www.freedesktop.org/wiki/Software/systemd/systemd-boot/)

#### Installation

1. On router computer, boot from the USB key
2. Change directories to where you put the image: `cd /livemnt/boot/data`
3. Convert from MBR to GPT
    * using parted: `parted /dev/sda mklabel gpt`
    * using gdisk: `gdisk /dev/sda`, then write the changes and exit
4. Using cgdisk, or some other gpt-enabled partitioning tool:
    * create an ESI partition (`ef00`) of between 64 and 256MB, and label it BOOT
    * create a linux partition (`8300`) using the rest of the disk, and label it ROOT
5. Format the partitions:
    * `mkfs.vfat -F 32 -n BOOT /dev/sda1` (`-F 32` => FAT32, `-n BOOT` => set volume name to `BOOT`)
    * `mkfs.ext4 -L ROOT /dev/sda2` (`-L ROOT` => set volume name to `ROOT`)
6. Set up the EFI boot partition:
    * Mount to `BOOT` partition: `mount /dev/sda` /mnt/bootfs`
    * Create the EFI boot image directory: `mkdir -p /mnt/bootfs/EFI/BOOT`
    * If doing a bare kernel install:
        * Copy the kernel into the boot image directory: `cp /livemnt/boot/data/lede-efi/lede-...-x86-64-vmlinuz /mnt/bootfs/EFI/BOOT/bootx64.efi`
        * Create the EFI startup script to pass the necessary kernel params: `echo 'fs:\\EFI\\BOOT\\bootx64.efi root=/dev/sda2 rootfstype=ext4 rootwait console=tty0 noinitrd" > /mnt/bootfs/startup.nsh`
    * If doing a systemd-boot install:
        * Copy the systemd-boot files into the boot image directory: `cp -R /livemnt/boot/data/lede-efi/systemd-boot/* /mnt/bootfs/`
7. Set up the root partition:
    * Mount the `ROOT` partition: `mount /dev/sda2 /mnt/rootfs`
    * Mount the rootfs image: `mount -o loop /livemnt/boot/data/lede-...-x86-64-rootfs-ext4.img /mnt/ledefs`
    * Copy all the files over: `rsync -avxHAWX --numeric-ids --info=progress2 /mnt/ledefs/ /mnt/rootfs/`
8. Reboot the system
9. **NOTHING WILL DISPLAY WHEN THE SYSTEM STARTS BOOTING.  DON'T PANIC: LEDE 17.0x KERNELS DON'T HAVE EFI FRAMEBUFFER CONFIGURED.**
    * See [my notes on building the openwrt kernel](/hobbies/openwrt/kernel.html) if you want to include EFI framebuffer support

References:
* [Guide to installing OpenWRT on MinnowBoards](http://elinux.org/Minnowboard:MinnowMaxDistros#OpenWrt)
* [ArchLinux UEFI boot guide](https://wiki.archlinux.org/index.php/GNU_Parted#UEFI.2FGPT_examples)
* [OpenWRT documentation for x86](https://wiki.openwrt.org/inbox/doc/openwrt_x86)

### Configuration

1. From a system connected to the LAN port of the router, ssh into `192.168.1.1`
2. Set a password: `passwd`
3. If the router is being setup with a private network connect to the WAN port, specifically if the upstream network is `192.168.1.x`:
    * change the router's IP to a different subnet: `uci set network.lan.ipaddr=192.168.2.1 && uci commit network && reload_config`
    * disconnect and reconnect the _client_ computer to the router to get an ip on the new network (either physically, or run `sudo ifdown eth0 && sudo ifup eth0`)
    * In the rest of the guide, substitute `192.168.2.x` for references to `192.168.1.x`
4. Install the `luci` web interface: `opkg update; opkg install luci`
5. Open a web browser, and type `http://lede/` into the navigation bar
6. System => System
    * Set a `Hostname`
    * Set the `Timezone`
7. Network => DHCP and DNS
    * Set a `Local server`
    * Set a `Local domain` (should generally have the same value as `Local server`)

References:
* [LEDE Quick Start guide](https://lede-project.org/docs/guide-quick-start/start)

### Bonus Round

* [LuCi https certificate](https://lede-project.org/docs/user-guide/getting-rid-of-luci-https-certificate-warnings)
* [Smart Queue Management (SQM)](https://lede-project.org/docs/user-guide/sqm) - minimizing [bufferbloat](https://www.bufferbloat.net/projects/bloat/wiki/What_can_I_do_about_Bufferbloat/)
* luci-app-adblock
* luci-app-ddns

#### Cisco/Fortigate/etc IPSEC VPN

* System => Software
  1. Download and install package: vpnc
  2. ssh into the router, and edit `/etc/vpnc/default.conf`
  3. Download vpnc-script from [here](http://git.infradead.org/users/dwmw2/vpnc-scripts.git/blob_plain/HEAD:/vpnc-script) and save it as /etc/vpnc/vpnc-script (and be sure to set +x on it)
  4. Create an init script to manage the vpnc service:
  ```
  #!/bin/sh /etc/rc.common

  START=50
  STOP=50
  USE_PROCD=1
  #PROCD_DEBUG=1

  start_service() {
    procd_open_instance vpnc
    procd_set_param command  /usr/sbin/vpnc --non-inter --no-detach
    procd_set_param watch network.interface

    # respawn automatically if something died
    # if process dies sooner than respawn_threshold, it is considered
    # crashed and after 5 retries the service is stopped
    procd_set_param respawn ${respawn_threshold:-3600} ${respawn_timeout:-5} ${respawn_retry:-5}

    procd_set_param file /etc/vpnc/default.conf
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
  }

  stop_service() {
    /usr/sbin/vpnc-disconnect
  }

  reload_service() {
    procd_send_signal vpnc stop
    procd_send_signal vpnc start
  }
  ```


#### OpenConnect SSL VPN

* https://wiki.openwrt.org/doc/howto/openconnect-setup
