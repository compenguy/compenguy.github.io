## Building a Custom OpenWRT kernel

### Building the kernel
* [LEDE developer guide for building images](https://lede-project.org/docs/guide-developer/use-buildsystem)
* [EFI OpenWRT on MinnowMax](http://elinux.org/Minnowboard:MinnowMaxDistros#OpenWrt)
* [Open bug report](https://bugs.lede-project.org/index.php?do=details&task_id=515) against lede project for missing EFI Framebuffer
* [Gentoo guide to building an EFI stub kernel](https://wiki.gentoo.org/wiki/EFI_stub_kernel)

To build a custom kernel in lede:

```bash
$ git clone https://git.lede-project.org/source.git lede
$ cd lede
$ ./scripts/feeds update -a
$ ./scripts/feeds install -a
$ make defconfig
$ make menuconfig
# set target system to x86
# set subtarget to x86_64
# set Target Images => Kernel partition size: 64MB
# Save
# Exit
$ make kernel_menuconfig CONFIG_TARGET=subtarget
# set 64-bit kernel
# Processor Type and Features => Processor family => Check Core 2/newer Xeon
# Processor Type and Features => Check x86 architectural random number generator
# Processor Type and Features => Check Symmetric Multiprocessing support
# Power Management and ACPI Options => Check Cpuidle Driver for Intel Processors
# Device Drivers => Serial ATA and Parallel ATA drivers (libata) => Check AHCI SATA support
# Device Drivers => Serial ATA and Parallel ATA drivers (libata) => Check Platform AHCI SATA support
# Device Drivers => Network device support => Ethernet driver support => Check Intel(R) PRO/1000 Gigabit Ethernet support
# Device Drivers => Network device support => Ethernet driver support => Check Intel(R) PRO/1000 PCI-Express Gigabit Ethernet support
# Device Drivers => Network device support => Ethernet driver support => Check Intel(R) 82575/82576 PCI-Express Gigabit Ethernet support
# Cryptographic API => Check AES cipher algorithms (AES-NI)
# disable support for various Vmware and MS virtualization technologies
# Save
# Exit
$ make V=99
```

### Creating an initrd
* Create an [initramfs](http://jootamam.net/howto-initramfs-image.htm):
```bash
mkdir -p initramfs/{bin,sbin,etc,proc,sys,newroot}
touch initramfs/etc/mdev.conf
wget http://jootamam.net/initramfs-files/busybox-1.10.1-static.bz2 -O - | bunzip2 > initramfs/bin/busybox
chmod +x initramfs/bin/busybox
ln -s busybox initramfs/bin/sh
touch initramfs/init
chmod +x initramfs/init
```
* `initramfs/init` contents (be sure to fill in the correct root location under `#Defaults`):
```bash
#!/bin/sh

#Defaults
init="/sbin/init"
root="/dev/sda2"

#Mount things needed by this script
mount -t proc proc /proc
mount -t sysfs sysfs /sys

#Disable kernel messages from popping onto the screen
echo 0 > /proc/sys/kernel/printk

#Clear the screen
clear

#Create all the symlinks to /bin/busybox
busybox --install -s

#Create device nodes
mknod /dev/null c 1 3
mknod /dev/tty c 5 0
mdev -s

#Function for parsing command line options with "=" in them
# get_opt("init=/sbin/init") will return "/sbin/init"
get_opt() {
	echo "$@" | cut -d "=" -f 2
}

# Process command line options
for i in $(cat /proc/cmdline); do
	case $i in
		root\=*)
			root=$(get_opt $i)
			;;
		init\=*)
			init=$(get_opt $i)
			;;
	esac
done

# Mount the root device
mount "${root}" /newroot

# Check if $init exists and is executable
if [[ -x "/newroot/${init}" ]] ; then
	#Unmount all other mounts so that the ram used by
	#the initramfs can be cleared after switch_root
	umount /sys /proc
	
	#Switch to the new root and execute init
	exec switch_root /newroot "${init}"
fi

# This will only be run if the exec above failed
echo "Failed to switch_root, dropping to a shell"
exec sh
```
* and finally, write the entire directory tree to a cpio archive:
```bash
cd initramfs
find . | cpio -H newc -o > ../initramfs.cpio
cd ..
gzip initramfs.cpio
```

### EFI Booting
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
* Format the partitions:
```bash
mkfs.vfat -F 32 -n BOOT /dev/sda1
mkfs.ext4 -L ROOT /dev/sda2
```
  - mkfs.vfat
    + `-F 32` FAT size 32 (e.g. FAT32)
    + `-n BOOT` set volume name to BOOT
  - mkfs.ext4
    + `-L ROOT` set volume label to ROOT
* Copy the data from the lede rootfs image onto your usb stick:
```bash
mount -o loop /livemnt/data/lede-...-x86-64-rootfs-ext4.img /mnt/ledefs
rsync -avxHAWX --numeric-ids --info=progress2 /mnt/ledefs/ /mnt/rootfs/
```
* to run qemu using a disk image:
```bash
qemu-system-x86_64 -m 1024 --bios /usr/share/ovmf/OVMF.fd -net none -hda ./sda.img -snapshot
```
  - `-m 1024` 1024 MB of RAM
  - `--bios /usr/share/ovmf/OVMF.fd` boot the OVM UEFI image
  - `-net none` disable ethernet so that it skips attempting to PXE boot
  - `-hda ./sda.img` to select the disk image to use
  - `-snapshot` write any disk modifications to temp files rather than modifying the disk image
