## Linuxing Around

Whatever the heck that means.

We all know linux is great... [it does infinite loops in 5 seconds](https://en.wikipedia.org/wiki/Portal:Linux/Selected_quote/4).

### Vim
Nobody *knows* vim.  Everyone who uses it is *learning* vim.

And on my quest for vimprovement...
* https://sanctum.geek.nz/arabesque/vim-anti-patterns/

### Bash
Bash is good for some stuff.  And then there's [Bash on Balls](https://github.com/jneen/balls), the web framework in bash that nobody asked for.

Some links to help me on my way:
* http://samrowe.com/wordpress/advancing-in-the-bash-shell/

### Lede/OpenWRT
Yeah, I built my own router.  Sort of.  I don't actually have it running yet.  On my path to VPNlightenment, I've picked up some stuff.

Getting Lede:
* [LEDE releases](https://downloads.lede-project.org/releases/)
* [LEDE Quick Start guide](https://lede-project.org/docs/guide-quick-start/start)

Basic installation instructions (MBR):
* [Guide to using OpenWRT combined images on x86-64](https://we.riseup.net/lackof/openwrt-on-x86-64#using-the-combined-images)
* [OpenWRT documentation for x86](https://wiki.openwrt.org/inbox/doc/openwrt_x86)

Probably want to build the kernel from git lede's git, though, since [EFI framebuffer support is not in the 17.01.0 release](http://www.mail-archive.com/lede-dev@lists.infradead.org/msg05989.html), although EFI stub support is.  Besides, it boots way faster if you turn off the stupid VM guest drivers and other crap that's not needed.
* [LEDE developer guide for building images](https://lede-project.org/docs/guide-developer/use-buildsystem)

The short version:
```bash
$ scripts/feeds update -a
$ scripts/feeds install -a
$ make menuconfig
# set target system to x86
# set subtarget to x86_64
# Save
# Exit
$ make defconfig
$ make menuconfig
# modify package settings
# modify build system settings
# modify kernel modules
$ make kernel_menuconfig CONFIG_TARGET=subtarget
```

EFI booting Lede:
* [OpenWRT EFI Boot on Intel Minnowboards](http://elinux.org/Minnowboard:MinnowMaxDistros#OpenWrt)
* [ArchLinux UEFI boot guide](https://wiki.archlinux.org/index.php/GNU_Parted#UEFI.2FGPT_examples)
* either use physical disks, or do the following to to create disk images:
```bash
# dd if=/dev/zero of=~/file.img bs=1MiB count=1024
# losetup --find --show ~/file.img
/dev/loop0
# kpartx -v -a ~/file.img
add map loop0p1 (253:0): 0 524288 linear 7:0 2048
add map loop0p2 (253:1): 0 1570783 linear 7:0 526336
```
  - dd
    + `bs=1MiB` 1 MB block size
    + `count=1024` 1024 blocks of `blocksize` each, e.g. 1024 * 1MiB = 1GiB total disk size
  - losetup
    + `--find` find the first unused loop device
    + `--show` display the name of the assigned loop device
  - kpartx
    + `-v` verbose output
    + `-a` add partition mappings
* create GPT partitions with cgdisk (I'm sure I've managed it with cfdisk, but apparently cgdisk is specifically tailored to the task)
* GPT partition code for ESP partitions is `ef00`
* GPT partition code for linux filesystem partitions is `8300`
* `mkfs.vfat -F 32 -n BOOT /dev/sda1`
  - `-F 32` FAT size 32 (e.g. FAT32)
  - `-n BOOT` set volume name to BOOT
* `mkfs.ext4 -L ROOT /dev/sda2`
  - `-L ROOT` set volume label to ROOT
* `mount -o loop /livemnt/data/lede-...-x86-64-rootfs-ext4.img /mnt/ledefs`
* `rsync -avxHAWX --numeric-ids --info=progress2 /mnt/ledefs/ /mnt/rootfs/`
* to run qemu using a disk image:
```bash
$ qemu-system-x86_64 -m 1024 --bios /usr/share/ovmf/OVMF.fd -net none -hda ./sda.img -snapshot
```
  - `-m 1024` 1024 MB of RAM
  - `--bios /usr/share/ovmf/OVMF.fd` boot the OVM UEFI image
  - `-net none` disable ethernet so that it skips attempting to PXE boot
  - `-hda ./sda.img` to select the disk image to use
  - `-snapshot` write any disk modifications to temp files rather than modifying the disk image

### Books
I've been meaning to take a closer look at [The Linux Programming Interface](http://man7.org/tlpi/).
