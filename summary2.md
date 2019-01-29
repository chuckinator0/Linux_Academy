# Saturday 2 February 2019

## Linux Academy

### Install a Boot Manager

Legacy GRUB

* Grand Unified Bootloader
* `boot.img` (first 512 bytes) -> `core.img` to find partition with `/boot/`
* `/boot/grub/` contains `grub.conf` (Redhat) or `menu.lst` (Debian) and `device.map`
* now linux kernel gets read and system boots
* install grub with `grub-install <device>`
  * device can be `/dev/sda` or `/dev/hd0` or `'(hd0)'`. Use `findmnt /boot` to find the device
* running `grub-install` is dangerous on a system that is already up and running
* the `grub` shell treats `\boot` as its root directory

GRUB2

* Master Boot Record (with legacy GRUB)
  * supported 26 partitions
  * partition limited to 2 TB
* GPT (GUID Partition Table)
  * supports 128 partitions, around a ZB each
  * needs `UEFI` (Unified Extensible Firmware Interface) to boot
    * replacement for traditional BIOS
    * requires 64 bit OS
    * prevents unauthorized OS from booting on the system
* UEFI BIOS -> master boot loader `boot.img` --> GPT header --> partition entry array --> `core.img` --> `/boot/efi` on a vfat or FAT32 volume (no other volume types) -> `/boot/grub2` which has `grubenv` and `themes` files
* On Redhat, we use `grub2-<command>`, whereas on Debian we use `grub-<command>`
* do not directly edit files in the `/boot/grub` directory
* `grub2-editenv list` to view default boot entry for the grub configuration file
* `grub2-mkconfig` cretaes or updates a `/boot/grub2/grub.cfg` file
* `update-grub` updatesd the grub2 configuration after changes to `\etc\default\grub` have been made

Interact With the GRUB Bootloaders

* 

----
----

## Miscellaneous