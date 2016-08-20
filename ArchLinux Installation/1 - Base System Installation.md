# Base System Installation

***DO NOT USE THESE NOTES BLINDLY.***  
***SOME CONFIGURANTIONS ARE PERSONAL AND PROBABLY OUTDATED.***

These notes were created for my desktop and notebook (Asus UX51VZ) installations.  
The desktop uses a **single SSD** and the notebook uses **two SSD's on RAID 0 with dm-crypt+LUKS** configuration.  

## Create bootable USB

#### Windows

This method does not require any workaround and is as straightforward as `dd` under Linux.  
Just download the Arch Linux ISO, and with local administrator rights use the USBwriter utility to write to the USB. 

<pre>
Download: <a href="http://sourceforge.net/projects/usbwriter/">http://sourceforge.net/projects/usbwriter/</a>
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#Using_USBwriter
</sup></sub>

#### Linux

Find out the name of the USB drive with `lsblk`.  
Do **not** append a partition number and make sure the drive is **not** mounted.  

<pre>
dd bs=4M if=<b>/path/to/archlinux.iso</b> of=/dev/<b>sdX</b> && sync
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#In_GNU.2FLinux
</sup></sub>

## Restore partitions on the USB

The procedure to create a bootable USB will create a partition on the USB.  
This methods will help restore the USB to a single primary partition.

#### Windows

<pre>
Open an elevated command prompt
Type <b>diskpart</b>
Type <b>List disk</b>
Find the disk number of the USB drive (it should be obvious going by the size)
Type <b>Select disk X</b>
Type <b>List partition</b>
Type <b>Select partition X</b>
Type <b>Delete partition</b>
Type <b>Create partition primary</b>
Type <b>Exit</b>
</pre>

<sub><sup>
References:
http://superuser.com/questions/536813/how-to-delete-a-partition-on-a-usb-drive
</sup></sub>

#### Linux

<pre>
Open a terminal and type <b>sudo su</b> to enter root
Type <b>fdisk -l</b> and note the USB drive letter
Type <b>fdisk /dev/sdX</b>
Type <b>d</b> to proceed to delete a partition
Type <b>1</b> to select the 1st partition and press enter
Type <b>d</b> to proceed to delete another partition (fdisk should automatically select the second partition)
Type <b>n</b> to make a new partition
Type <b>p</b> to make this partition primary and press enter
Type <b>1</b> to make this the first partition and then press enter
Press enter to accept the default first cylinder
Press enter again to accept the default last cylinder
Type <b>w</b> to write the new partition information to the USB key
Type <b>umount /dev/sdX</b>
Type <b>mkfs.vfat -F 32 /dev/sdX</b> to create a FAT32 filesystem. (Replace the FS type as prefered)
</pre>

<sub><sup>
References:
http://dottheslash.wordpress.com/2011/11/29/deleting-all-partitions-on-a-usb-drive/
</sup></sub>

## Install base system

#### Check and load available keyboard layouts

<pre>
ls /usr/share/kbd/keymaps/i386/qwerty
</pre>

<pre>
loadkeys pt-latin9
</pre>

In my case there are two keymaps available, `pt-latin1` and `pt-latin9`, the differences between them, can be seen [here](https://www.cs.tut.fi/~jkorpela/latin9.html).  Just keep in mind that the ISO Latin 9 is more recent.  

#### (Optional) Partition plan

Before any change, get to know the details of each recommended partition.

> https://wiki.archlinux.org/index.php/partitioning#Partition_scheme  
> http://en.wikipedia.org/wiki/Disk_partitioning#Benefits_of_multiple_partitions

I personally don't use any partition schema because it reduces the total space available on the SSD drives. Instead, I have an external HDD with a full partition backup of my system in case anything goes wrong.

Although, recently I changed my **notebook** primary partition (`/dev/md125p3`) to an extended partition (still `/dev/md125p3`) to accommodate a  `100MiB /boot` logical partition (`/dev/md125p5`) and a encrypted root logical partition (`/dev/md125p6`).  
This was done because I added a dm-crypt layer to my system as described below. 

##### Check existing partitions

<pre>
fdisk -l
</pre>

##### (1) Change MBR partitions

<pre>
cfdisk /dev/<b>sdX</b>
</pre>

##### (2) Change GPT partitions

<pre>
cgdisk /dev/<b>sdX</b>
</pre>

##### (Optional) SSD Alignment

It is also worth mentioning, that once the partitions have been set it is important that they are aligned.
The alignment can be checked using `parted`. If they are not aligned, like already happen to me, [this procedure might help.](https://github.com/Tenza/configurations/blob/master/ArchLinux%20Installation/TODO.md#ssd-alignment)

<pre>
parted /dev/md125
(parted) align-check opt 1                                                
1 aligned
(check reminder partitions)
</pre>

#### Format the filesystem

After the partitions have been set, the filesystem has to be formatted. I will use the EXT4 filesystem.

Also note that the `discard` flag is used, it will attempt to discard blocks at mkfs time (discarding blocks initially is useful on solid state devices and sparse / thin-provisioned storage). When the device advertises that discard also zeroes data (any subsequent read after the discard and before write returns zero), then mark all not-yet-zeroed inode tables as zeroed. This significantly speeds up filesystem initialization. This is set as default. 

##### Desktop

<pre>
mkfs.ext4 -v -L Arch -E discard /dev/<b>sda1</b>
</pre>

##### Notebook

With a RAID configuration, additional flags should be used in order to correctly format and align the raid array.  
Use the following command to find out the chunk size (128k in my case).  
The blocksize is normally 4k, it is used for somewhat large files.  

<pre>
mdadm -E /dev/<b>md127</b>
</pre>

* Stride = (chunk size/block size). 
 * Stride = 128 / 4 = 32
* Stripe-width = (number of physical data disks * stride). 
 * Stripe-width = 2 * 32 = 64

Lastly the size of blocks in bytes has to be specified. Valid block-size values are 1024, 2048 and 4096 bytes per block. If omitted, block-size is heuristically determined by the filesystem size and the expected usage of the filesystem. But since I already used a 4k block size to determine the stride, I will use 4096 bytes.

> I also tried letting the `mkfs` use its defaults (`mkfs.ext4 -v -L Arch /dev/md125p3`) in order to determine the above parameters. And it correctly guessed everything, so if there is any doubt about something, just trust the defaults. 

<sub><sup>
References:  
https://wiki.debian.org/SSDOptimization#Mounting_SSD_filesystems  
http://blog.nuclex-games.com/2009/12/aligning-an-ssd-on-linux/  
</sup></sub>

###### (1) Format without dm-crypt

Simply format the partition, with the additional flags, without encryption.

<pre>
mkfs.ext4 -v -L Arch -b 4096 -E stride=32,stripe-width=64,discard /dev/<b>md125p3</b>
</pre>

###### (2) Format with dm-crypt

There are multiple ways to enable encryption. [This table](https://wiki.archlinux.org/index.php/disk_encryption#Comparison_table) might help to decide on wish to use. Also, if necessary, [prepare the disk](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation) by overwriting the entire partition/drive with random data, this will enable protection from cryptographic attacks or unwanted file recovery, this data is ideally indistinguishable from data later written by dm-crypt. 

Bellow I will describe the setup of `dm-crypt + LUKS` on a single partition layout **without** `LVM`. 
Keep in mind that I choose not to create a `LVM` volume because I will not take advantage of any of its features, and also that I have a separate `100MiB /boot` logical partition, that I **don't** enable encryption.

First setup a new dm-crypt device in LUKS encryption mode, mapped to the unencrypted partition.  
This will prompt for a passphrase that will need to be typed at each boot into Arch.

Note that this command uses the default parameters of LUKS, [click here](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode) to view the what is being used by default.

<pre>
cryptsetup -v luksFormat /dev/md125p6
</pre>

Than open and set the name of the mapped device.  
The name `ArchCrypt` will be seen when dm-crypt asks for your passphrase at boot.

<pre>
cryptsetup open /dev/md125p6 ArchCrypt
</pre>

Format the filesystem of the mapped `/ArchCrypt` partition with the flags desired.

<pre>
mkfs.ext4 -v -L ArchRoot -b 4096 -E stride=32,stripe-width=64,discard <b>/dev/mapper/ArchCrypt</b>
</pre>

Also format the logical `/boot` partition.

<pre>
mkfs.ext4 -v -L ArchBoot -b 4096 -E stride=32,stripe-width=64,discard <b>/dev/md125p5</b>
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/disk_encryption#Comparison_table  
https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Simple_partition_layout_with_LUKS  
https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode 
</sup></sub>

#### Create folders and mount partitions

Create a mountpoint for the installation of the system.  

##### Desktop

<pre>
mount -t ext4 /dev/<b>sda1</b> /mnt
</pre>

##### Notebook

###### (1) Mount without dm-crypt

<pre>
mount -t ext4 /dev/<b>md125p3</b> /mnt
</pre>

###### (2) Mount with dm-crypt

<pre>
mount -t ext4 /dev/mapper/ArchCrypt /mnt
</pre>

Additionally create and mount the unencrypted `/boot` partition.

<pre>
mkdir /mnt/boot
mount -t ext4 /dev/md125p5 /mnt/boot
</pre>

#### SWAP

A SWAP is used for moving data in memory to disk and to enable the hibernate functionality. In this case, I will be using a swap file instead of a partition because it is easer to resize and having a dedicated partition for this does not seem useful. Note that the file should have at least the same size of the physical RAM, although for me, I do not plan on hibernate with my RAM full, so I will give less space.

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Swap#Swap_file
</sup></sub>

##### (1) Create SWAP file

<pre>
fallocate -l 6G /mnt/swapfile  
chmod 600 /mnt/swapfile  
mkswap -L ArchSwap /mnt/swapfile  
swapon /mnt/swapfile  
swapon -s
</pre>

##### (2) Create SWAP partition

<pre>
mkswap -L ArchSwap /mnt/<b>sdaX</b>
swapon /mnt/<b>sdaX</b>  
swapon -s
</pre>

#### Check network

Before the installation of the base system, make sure that the network is working.
If a wired connection in being used, try the ping command because DHCP should be activated by default.

<pre>
ping -c 3 google.com
</pre>

| Switch | Description | 
| --- | --- | 
| -c X | Number of data packets to send |

##### (Optional) Activate wireless connectivity

In the absence of a wired network use the following command to activate wireless.

<pre>
wifi-menu
</pre>

#### Install base system

This is the command that will download and install the ArchLinux operating system. Keep in mind that only the `base` package is needed have a working system, but the `base-devel` package was also added because it will enable us to build packages to add to the system.

On the live system, all download mirrors are enabled, and sorted by their synchronization status and speed at the time the installation image was created. The higher a mirror is placed in the list, the more priority it is given when downloading a package. You may want to edit the file accordingly, and move the geographically closest mirrors to the top of the list, although other criteria should be taken into account. The pacstrap tool used in the next step also installs a copy of the file to the new system, so it is worth getting right.

<pre>
pacstrap -i /mnt base base-devel
</pre>

| Switch | Description | 
| --- | --- | 
| -i | Ensures prompting before package installation. With the base group, the first initramfs will be generated and installed to the new system's boot path; double-check output prompts ==> Image creation successful for it. |

> If there is a warning regarding `mandb` not being able to set the locale, that is normal because the locale will only be set inside the system (using chroot). Once the desired locales are enabled and `locale-gen` generates the locales, it will be solved. [This is actually flagged as a bug that might be solved by now](https://bugs.archlinux.org/task/49426).

> Since the image that I download is always the most up-to-date, I expect the mirrors to be in a good state, so I don't do any mirror optimization at this point.

#### Install bootloader

The bootloader to be installed depends on the partition style and the board firmware type.

Example 1, if the board has UEFI, and Windows 7 installed, it is safe to assume that the CMS mode is enabled and the partition style is MBR. Knowing this, a bootloader for a BIOS system should be used.

Example 2, if the board has UEFI, and Windows 8 installed, it is safe to assume that the CMS mode is disabled and the partition style is GPT. Knowing this, a bootloader for a UEFI system should be used.

> The NT bootloader is located on a `100MB "System Reserved"` partition. This cannot be erased even with a different bootloader because other bootloaders cannot directly start the OS. They have to chainload with the NT bootloader in order to start Windows.

[Also keep in mind that there are limitations between the two.](https://wiki.archlinux.org/index.php/Windows_and_Arch_Dual_Boot#Bootloader_UEFI_vs_BIOS_limitations)

#### Install GRUB2 for BIOS-MBR system

<pre>
pacstrap -i /mnt grub-bios 
</pre>

Using `pacstrap` or `pacman -S grub` is the same thing. But the former will get the correct package if the name changes.

#### Generate filesystem table

<pre>
genfstab -L -p /mnt >> /mnt/etc/fstab
</pre>

| Switch | Description | 
| --- | --- | 
| -L | Indicates genfstab to use Labels. |
| -U | Indicates genfstab to use UUIDs. |
| -p | Avoids printing pseudofs mounts. |

<sub><sup>
References:
http://unix.stackexchange.com/questions/124733/what-does-genfstabs-p-option-do
</sup></sub>

#### (Optional) Change filesystem table flags for SSD optimization.

The following flags on the file fstab are used on SSD disks. Start by checking if the kernel knows about the SSDs. 

<pre>
/sys/block/sd?/queue/rotational
</pre>

The `discard` flag within the fstab file is used to enable continuously TRIM support. Unfortunately, this is known to cause data corruption problems. Even more frequent problems arise on top of a RAID configuration. [In here are some good arguments against the use of TRIM, and why it should be avoided.](http://www.spinics.net/lists/raid/msg40916.html)
Moreover, the kernel [even blacklists some devices know to cause problems.](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/drivers/ata/libata-core.c#n4270)

<pre>
nano /mnt/etc/fstab
</pre>

According to the [linux fstab page](http://man7.org/linux/man-pages/man5/fstab.5.html), the `defaults` flag sets the following options: `rw, suid, dev, exec, auto, nouser, async`, check what each of these flags do on the [linux mount page](http://man7.org/linux/man-pages/man8/mount.8.html).

The `noatime` flag eliminates the need of the system to make writes to the filesystem for files which are simply being read.  
Also make sure the SWAP file is not on `/mnt` and rather on `/`.

My configuration, without any storage disks, has the following parameters.

<pre>
/dev/sda1               /               ext4            rw,noatime,stripe=64,data=ordered      0 1
/swapfile               none            swap            defaults,noatime      0 0
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Solid_State_Drives#noatime_Mount_Flag  
https://wiki.archlinux.org/index.php/Solid_State_Drives#Enable_TRIM_by_Mount_Flags
</sup></sub>

#### Enter change-root

Finally run `arch-chroot` to enter the freshly installed system.  
From this moment forward, there is no need to refer to the `/mnt` mountpoint.

<pre>
arch-chroot /mnt
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Change_root
</sup></sub>

#### (Optional) (RAID) Create file to assembly RAID arrays and enable the mdadm mkinitcpio hook

This is required for the RAID 0 configuration. The module `mdadm_udev` will use the information generated here to assemble the array on boot. Also, note that even if the wiki says that `mdraid` is used for fake raid systems, Intel advises the use of `mdadm` for their boards. Start by finding out where the RAID array is located.

<pre>
mdadm --misc --help
mdadm --detail-platform
mdadm --examine /dev/<b>mdX</b>
</pre>

In my case the array is located on md127, and the md125 and md126 are the members of the Intel Matrix Storage Manager (imsm) controler. Now simply get the identity of the members with the help of `mdadm`, and add them to the file `/etc/mdadm.conf` in order to assemble the array on boot. These instructions can be read in the `/etc/mdadm.conf` file itself.

<pre>
mdadm --examine --scan >> /etc/mdadm.conf
</pre>

Now enable the `mdadm_udev` hook upon boot, **in the correct order** in order to assemble the RAID array. 

> Although the wiki advocates for the use of `mdadm_udev`, I also tried using the module `mdadm`, but unfortunatly it generated a segmentation fault, and my system did not boot.

<pre>
nano /etc/mkinitcpio.conf  
</pre>

<pre>
HOOKS="... block <b>mdadm_udev</b> filesystems ..."
</pre>

<sub><sup>
References:  
http://www.intel.com/content/www/us/en/intelligent-systems/software/rst-linux-paper.html
https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz  
https://wiki.archlinux.org/index.php/RAID#Installing_Arch_Linux_on_RAID  
https://forums.gentoo.org/viewtopic-t-888520.html
</sup></sub>

#### (Optional) (Encryption) Enable the dm-crypt mkinitcpio hook

If encryption was setup, the hook needs to be loaded on boot. Add the `encrypt` hook to mkinitcpio, **in the correct order**. It should be after the `mdadm_udev` if a raid array is present, because the array needs to be assembled before the encryption layer can be loaded.

<pre>
nano /etc/mkinitcpio.conf  
</pre>

<pre>
HOOKS="... block mdadm_udev <b>encrypt</b> filesystems ..."
</pre>

Other hooks might be necessary depending on the physical computer and encryption setups.  
For example, for USB keyboards, the hook `keyboard` needs to be added to make sure it works early in userspace. Also don't forget that this hook needs to be set before the `encrypt`, otherwise it will not be possible to type the passphrase.

#### (Optional) Create initial ramdisk

This was already created when the base system was installed.  
This is only needed if any modules were added, like adding the `mdadm` or `encrypt` module.

<pre>
mkinitcpio -p linux
</pre>

| Switch | Description | 
| --- | --- | 
| -p | Specifies a preset to utilize; most kernel packages provide a related mkinitcpio preset file, found in `/etc/mkinitcpio.d` (e.g. `/etc/mkinitcpio.d/linux.preset` for linux). A preset is a predefined definition of how to create an initramfs image instead of specifying the configuration file and output file every time. |

<sub><sup>
References:
https://wiki.archlinux.org/index.php/mkinitcpio
</sup></sub>

##### (Optional) (Encryption) Enable dm-crypt at boot

In order to unlock the encrypted root partition at boot, the following kernel parameters need to be set.

<pre>
nano /etc/default/grub
</pre>

<pre>
GRUB_CMDLINE_LINUX="cryptdevice=/dev/md125p6:ArchCrypt root=/dev/mapper/ArchCrypt"
</pre>

The kernel parameters are added to `GRUB_CMDLINE_LINUX` are always effective, and the  `GRUB_CMDLINE_LINUX_DEFAULT` are effective ONLY during normal boot (NOT during recovery mode). Since these hooks are an integral part of the system setup, they need to be always executed.

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Dm-crypt/System_configuration#Boot_loader
</sup></sub>

#### Configure GRUB2

Even if no kernel parameters were added to `/etc/default/grub` the grub configuration file still needs to be generated. This command will essencially build the file `/boot/grub/grub.cfg` that contains the menu used by the bootloader.

<pre>
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

The `grub-mkconfig` command is equal to `update-grub` on Ubuntu, that is just a wrapper.  

> Note that if `os-prober` was already installed in order to dualboot, it might not detect other OS's installations until the system is rebooted to a non-live environment.

#### Install GRUB2 to the MBR

This will install the bootloader to the first sector of the disk.  
Note that it is possible to install to a partition, but is not recommended.  
Also, this will be needed if the formatted partition contained `/etc/default/grub` or `/etc/grub.d/`.

| Switch | Description | 
| --- | --- | 
| --target=i386-pc | instructs grub-install to install for BIOS systems only. It is recommended to always use this option to remove ambiguity in grub-install. |

##### Desktop  
<pre>
grub-install --target=i386-pc --recheck /dev/<b>sda</b>
</pre>

##### Notebook  
<pre>
grub-install --target=i386-pc --recheck /dev/<b>md125</b>
</pre>

#### Change the root password

<pre>
passwd
</pre>

#### (Optional) Install necessary wireless drivers

Without a wired connection, install the following packages to enable access to `wifi-menu` after the reboot.

<pre>
pacman -S iw wpa_supplicant dialog wpa_actiond
</pre>

#### Exit change-root and reboot the system

Finally exit change-root, reboot and remove the USB drive. The system should now be properly installed.

<pre>
exit
systemctl reboot
</pre>
