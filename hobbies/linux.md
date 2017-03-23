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
* https://downloads.lede-project.org/releases/
* https://lede-project.org/docs/guide-quick-start/start

Basic installation instructions (MBR):
* https://we.riseup.net/lackof/openwrt-on-x86-64#using-the-combined-images
* https://wiki.openwrt.org/inbox/doc/openwrt_x86

Probably want to build the kernel from git lede's git, though, since [EFI framebuffer support is not in the 17.01.0 release](http://www.mail-archive.com/lede-dev@lists.infradead.org/msg05989.html), although EFI stub support is.  Besides, it boots way faster if you turn off the stupid VM guest drivers and other crap that's not needed.
* https://lede-project.org/docs/guide-developer/use-buildsystem

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
* http://elinux.org/Minnowboard:MinnowMaxDistros#OpenWrt
* https://wiki.archlinux.org/index.php/GNU_Parted#UEFI.2FGPT_examples
* create GPT partitions with cgdisk (I'm sure I've managed it with cfdisk, but apparently cgdisk is specifically tailored to the task)
* GPT partition code for ESP partitions is `ef00`
* GPT partition code for linux filesystem partitions is `8300`
* `mkfs.vfat -F 32 -n BOOT /dev/sda1`
* `mkfs.ext4 -L ROOT /dev/sda2`
* `mount -o loop /livemnt/data/lede-...-x86-64-rootfs-ext4.img /mnt/ledefs`
* `rsync -avxHAWX --numeric-ids --info=progress2 /mnt/ledefs/ /mnt/rootfs/`

### Books
I've been meaning to take a closer look at [The Linux Programming Interface](http://man7.org/tlpi/).
