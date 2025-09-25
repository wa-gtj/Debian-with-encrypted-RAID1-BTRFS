# Debian-with-encrypted-RAID1-BTRFS
Directions to install Debian with BTRFS Filesystem, full disk encryption and RAID1. 

Goal of these instructions is an installation of debian on a LUKS-encrypted BTRFS-Filesystem, making use of the BTRFS-inherent RAID-functionality.

## Prerequisites
- You need to have two storage media to set up RAID
- You need to prepare the Installation media. Please consult the debian wiki: https://www.debian.org/releases/stable/amd64/ch04s03.de.html

## Disk partitioning
Choose "advanced options" -> "Expert Install".
Choose Language, configure keyboard etc. according your needs. Follow through to the Point "Partition disks."
Choose "manual."
### Create new partition tables
Go down to the first disk (in my case "sda" (not the first partition!) and hit enter.
Select Yes and hit enter to create a new empty partition table. Confirm that you want to use gpt when running on an UEFI machine.

Do the same with the second disk ("sdb").
### Partitioning of the disks
#### EFI system partition (ESP)
Select the free space of sda and create a new partition.
Make it 1 GB and put it at the beginning of the free space.
Instead of using it as "Ext4 journaling file system", select "EFI System Partition." 
#### encrypted SWAP area
Again select "free Space" to create a new partition. 
If you need a Swap-area, make it 1,5 times the size of your memory.
Select to use it as "physical Volume for encryption."
If you're running it on a SSD, disable "erase data."
#### Partitions for Installation
For the third partition use the remainder of the disk. Again, choose "physical Volume for encryption."

On the second drive, also create a new partition, but for the size, deduct about 1 GB to leave enough free space to use for an ESP in case the first disk fails. Allocate the partition to the end of the disk.
Of course, select "physical volume for encryption" again.
### LUKS configuration
Go up and choose "Configure encrypted Volumes". Confirm that you want to write the changes to disk.

Do not select "create enrcypted Volumes", as we have already done so, but select "finish".Confirm to erase data.

Choose a strong encryption passphrase. Use the same for all volumes.
### Configuration of partitions
Now, you see three encrypted volumes (sda2_crypt, sda3_crypt and sdb1_crypt) above the first disk.

"sda2_crypt" will be our swap-area. Select it and hit enter.  Alter "use it as" to "swap area."

For "sda3_crypt", use btrfs journaling filesystem, mount point "/", mount options "noatime" and "compress". You can also use a label.

Do nothing to sdb1_crypt. We will manually make a RAID-Volume at once.
### Finish partitioning
Finish partitioning and write changes to disk.
Confirm that you want to configure encryption without separate "/boot" partition.

DO NOT install the base system yet.
## BTRFS - Configuration
Enter Terminal: Hit Ctrl + Alt + F2

show the currently mounted Devices with their mountpoints

```
df -h
```

### Adding Device to create RAID:

```
btrfs device add /dev/mapper/sdb1_crypt /target
btrfs balance start -dconvert=raid1 -mconvert=raid1 /target
```

to track progress:

```
btrfs balance status /target
```

The  Answer should be "no balance found", because it has already finished.

Confirm the new RAID details, when finished:

```
btrfs filesystem show
```
### temporary mount of filesystem
unmount filesystem and mount temporarily to create subvolumes

```
umount /target/boot/efi
umount /target/
mount /dev/mapper/sda3_crypt /mnt
cd /mnt 
```

### change subvolumes:

```
mv @rootfs @ 
```

ls

```
btrfs subvolume create @home @snapshots @/@log @/@cache
```

ls --> should show the 3 subvolumes, 

ls @ should show another two (@log and @cache) 

### mount subvolumes

mount root 

```
mount -o noatime,compress=zstd,subvol=@ /dev/mapper/sda3_crypt /target
```

create mountpoints

```
cd /target
mkdir -p ./boot/efi ./home ./.snapshots ./var/log ./var/cache
```

```
mount -o noatime,compress=zstd,subvol=@home /dev/mapper/sda3_crypt /target/home
mount -o noatime,compress=zstd,subvol=@snapshots /dev/mapper/sda3_crypt /target/.snapshots
mount -o noatime,compress=zstd,subvol=@/@log /dev/mapper/sda3_crypt /target/var/log
mount -o noatime,compress=zstd,subvol=@/@cache /dev/mapper/sda3_crypt /target/var/cache
mount /dev/sda1 /target/boot/efi
```

### Edit /etc/fstab

edit fstab according to current mounting

```
nano /target/etc/fstab
```

change second 0 to 1 for root subvolume, to 2 for home, snapshots, log and cache

Ctrl + O to write to file

Ctrl + X to exit

### Edit /etc/crypttab

make the boot loader ask only once for the passphrase to unlock all encrypted devices

https://unix.stackexchange.com/questions/392284/using-a-single-passphrase-to-unlock-multiple-encrypted-disks-at-boot

```
sed -i'.bak' -e 's/none/crypt_disks/g' -e 's/luks/luks,keyscript=decrypt_keyctl/g' /etc/crypttab
```

create line for /dev/sdb1 .  
sdb1_crypt UUID= crypt-disks luks,keyscript=decrypt_keyctl,discard

use the correct UUID! The one of the base partition, not the one of the mapped!

## Installing base system

Go back to the installer with Ctrl + Alt + F1.

Continue by installing the base system. Wwhen prompted confirm the preselected kernel.

When prompted for the installation of drivers, I choose to install only the relevant drivers for the target-system, not all generic drivers.

configure APT, choose https.

when an error occures, that you should insert the CDROM labeled "Debian Trixie..." to mount on /media/cdrom, enter busybox again (Ctrl + Alt + F2) and

```
umount /cdrom 
mount /dev/sr0 /target/media/cdrom
```

Go back to the installer and continue.

When prompted, choose to install no window manager. Instead, select the ssh-server.

## Prepare for installation of boot-loader
We have renamed the subvolume for the root of the filesystem, but there is still a reference to the old name in some files, which gets written to the bootloader-entry, when the bootloader is updated.
Not doing this, will prevent the system from booting, until the boot loader entry is edited manually in the busybox.

We can circumvent this, by editing some files. After that, update-initramfs writes the correct subvolume for root-filessystem to the \*.conf-files in /boot/efi/loader/entries.

Go to the Terminal again and run the following commands:

```
sed -i -e 's/@rootfs/@/g' /etc/kernel/cmdline  
sed -i -e 's/@rootfs/@/g' /usr/lib/kernel/cmdline  
sed -i -e 's/@rootfs/@/g' /proc/cmdline  
```

In my case, the third file contains the correct information and the second doesn't exist, which will result in error messages, which can be ignored.

## Bootloader installation
We will install "systemd-boot" as bootloader, as grub usually needs a seperate "/boot"-partition.

## Reboot
Reboot and enjoy!

