# Base System Installation

***DO NOT USE THESE NOTES BLINDLY.***  
***SOME CONFIGURANTIONS ARE PERSONAL AND PROBABLY OUTDATED.***

These notes were created for my desktop and notebook (Asus UX51VZ) installations.  
The desktop uses a **single SSD** and the notebook uses **two SSD's on a RAID 0** configuration.  
Both have **dualboot with Windows 7**, and since both use SSD drives I will use some recommend optimizations.  

## Create bootable USB

##### Windows

This method does not require any workaround and is as straightforward as `dd` under Linux.  
Just download the Arch Linux ISO, and with local administrator rights use the USBwriter utility to write to your USB flash memory. 

<pre>
Download: <a href="http://sourceforge.net/projects/usbwriter/">http://sourceforge.net/projects/usbwriter/</a>
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#Using_USBwriter
</sup></sub>

##### Linux

Find out the name of the USB drive with `lsblk`.  
Do **not** append partition a number and make sure the drive is not mounted.  

<pre>
dd bs=4M if=<b>/path/to/archlinux.iso</b> of=/dev/<b>sdX</b> && sync
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#In_GNU.2FLinux
</sup></sub>

## Restore partitions on the USB

##### Windows

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

##### Linux

<pre>
Open a terminal and type <b>sudo su</b> to enter root.
Type <b>fdisk -l</b> and note your USB drive letter.
Type <b>fdisk /dev/sdX</b>
Type <b>d</b> to proceed to delete a partition
Type <b>1</b> to select the 1st partition and press enter
Type <b>d</b> to proceed to delete another partition (fdisk should automatically select the second partition)
</pre>

Next we need to create the new partition:

<pre>
Type <b>n</b> to make a new partition
Type <b>p</b> to make this partition primary and press enter
Type <b>1</b> to make this the first partition and then press enter
Press enter to accept the default first cylinder
Press enter again to accept the default last cylinder
Type <b>w</b> to write the new partition information to the USB key
Type <b>umount /dev/sdX</b>
</pre>

The last step is to create the fat filesystem:

<pre>
Type <b>mkfs.vfat -F 32 /dev/sdX</b>
</pre>

<sub><sup>
References:
http://dottheslash.wordpress.com/2011/11/29/deleting-all-partitions-on-a-usb-drive/
</sup></sub>

## Install base system

Once we boot from the USB, we start by setting up the system.

##### Check avalable keyboard layouts

<pre>
ls /usr/share/kbd/keymaps/i386/qwerty
</pre>

##### Load keyboard layout

<pre>
loadkeys pt-latin9
</pre>

###### (Optional) Partition plan

Before any change, we should get to know the details of each recommended partition.

I personally don't use any partition scheme because it reduces the total space available, and because I have an external HDD with a full partition backup of my system in case anything goes wrong.

> https://wiki.archlinux.org/index.php/partitioning#Partition_scheme  
> http://en.wikipedia.org/wiki/Disk_partitioning#Benefits_of_multiple_partitions

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

###### (Optional) (RAID) Find out chunk and block size

This is needed to correctly format and align the raid array.  
Use the following command and find out the chunk size (128k in my case).  
The blocksize is normally 4096, it is used for somewhat large files.  

<pre>
mdadm -E /dev/<b>md127</b>
</pre>

* Stride = (chunk size/block size). 
 * Stride = 128 / 4 = 32
* Stripe-width = (number of physical data disks * stride). 
 * Stripe-width = 2 * 32 = 64

##### Format the filesystem with SSD and RAID optimizations

After the partitions have been set, format the filesystem to EXT4.  
In my case, the commands are the following:

Dektop:
<pre>
mkfs.ext4 -v -L Arch -E discard /dev/<b>sda1</b>
</pre>

Notebook:
<pre>
mkfs.ext4 -v -L Arch -b 4096 -E stride=32,stripe-width=64,discard /dev/<b>md125p4</b>
</pre>

<sub><sup>
References:  
https://wiki.debian.org/SSDOptimization#Mounting_SSD_filesystems  
http://blog.nuclex-games.com/2009/12/aligning-an-ssd-on-linux/  
</sup></sub>

##### Create folders and mount partitions

Now create a mount-point for the installation of the system.  
In my case, the commands are the following:

Dektop:
<pre>
mount /dev/<b>sda1</b> /mnt
</pre>

Notebook:
<pre>
mount /dev/<b>md125p4</b> /mnt
</pre>

(Optional) Folders for other partitions

With a partition schema, keep in mind the following commands:

<pre>
mkdir /mnt/boot /mnt/home /mnt/var
mount /dev/<b>sdaX</b> /mnt/boot
mount /dev/<b>sdaX</b> /mnt/home
mount /dev/<b>sdaX</b> /mnt/var
</pre>

##### SWAP

A SWAP is used for moving data in memory to disk and to enable the hibernate functionality. In this case, I will be using a swap file instead of a partition because it is easer to resize and having a dedicated partition for this does not seem useful. Note that the file should have at least the same size of the physical RAM, although for me, I do not plan on hibernate with my RAM full, so I will give less space.

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Swap#Swap_file
</sup></sub>

##### (1) Create SWAP file

<pre>
fallocate -l 6G /mnt/swapfile  
chmod 600 /mnt/swapfile  
mkswap /mnt/swapfile  
swapon /mnt/swapfile  
</pre>

##### (2) Create SWAP partition

<pre>
mkswap /mnt/<b>sda1</b>
swapon /mnt/<b>sda1</b>
</pre>

##### Check SWAP status

<pre>
swapon -s
</pre>

##### Check network

Before we start with the installation of the base system, we need to make sure that the network is working.
If we are using a wired connection, DHCP should be activated by default.

<pre>
ping -c 3 google.com
</pre>

###### (Optional) Activate wireless connectivity

If you do not have a wired network use the following command to activate wireless.

<pre>
wifi-menu
</pre>

##### Install base system

<pre>
pacstrap -i /mnt base base-devel
</pre>

The -i switch ensures prompting before package installation. With the base group, the first initramfs will be generated and installed to the new system's boot path; double-check output prompts ==> Image creation successful for it. 

##### Install bootloader

The bootloader to be installed depends on the partition style and the board firmware type.

Example 1, if we have a board with UEFI, and Windows 7 installed, we can assume that the CMS mode is enabled and the partition style is MBR. Knowing this, we should use a bootloader for a BIOS system.

Example 2, if we have a board with UEFI, and Windows 8 installed, we can assume that the CMS mode is disabled and the partition style is GPT. Knowing this, we should use a bootloader for a UEFI system.

https://wiki.archlinux.org/index.php/Windows_and_Arch_Dual_Boot#Bootloader_UEFI_vs_BIOS_limitations

---

The NT bootloader is located on a 100MB "System Reserved" partition. This cannot be erased even with a diferent bootloader because other bootloaders cannot directly start the OS. They have to chainload with the NT bootloader in order to start Windows.

---

##### Install GRUB2 for BIOS-MBR system

<pre>
pacstrap -i /mnt grub-bios 
</pre>

Using `pacstrap` or `pacman -S grub` is the same thing. But the former will get the correct package if the name changes.

##### Generate filesystem table

<pre>
genfstab -p /mnt >> /mnt/etc/fstab
</pre>

<sub><sup>
References:
http://unix.stackexchange.com/questions/124733/what-does-genfstabs-p-option-do
</sup></sub>

###### (Optional) Change filesystem table flags for SSD optimization.

<pre>
nano /mnt/etc/fstab
</pre>

> Add to the SSD disk parameters: `discard` to activate TRIM. **Only if you are sure that the SSD has support.**  
> Update the SSD disk parameters: `realatime` for `noatime` it eliminates the need by the system to make writes to the file system for files which are simply being read.  
> Also make sure the SWAP file is not on `/mnt` and rather on `/`  

Check if the kernel knows about SSDs:  
<pre>
/sys/block/sd?/queue/rotational
</pre>

Check for TRIM support:  
<pre>
hdparm -I /dev/sd? | grep TRIM
</pre>

My desktop, without the my storage disks, has the following parameters:
<pre>
/dev/sda1               /               ext4            rw,noatime,discard      0 1
/swapfile               none            swap            defaults,noatime,discard        0 0
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Solid_State_Drives#noatime_Mount_Flag  
https://wiki.archlinux.org/index.php/Solid_State_Drives#Enable_TRIM_by_Mount_Flags
</sup></sub>

##### Enter change-root

Once we change root we will be inside our system.  
From this moment, we no longer refer to the `/mnt` mount point .

<pre>
arch-chroot /mnt
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Change_root
</sup></sub>

###### (Optional) (RAID) Create file to assembly RAID arrays

This is required, for the RAID 0 configuration. The module `mdadm` will use the information generated here to assemble the array on boot. Also, note that even if the wiki says that `mdraid` is used for fake raid systems, Intel advises the use of `mdadm` for their boards. Start by finding out where your RAID array is located with:

<pre>
mdadm --detail-platform
mdadm -E /dev/<b>mdX</b>
</pre>

In my case the array is located on md127 and uses Intel Matrix Storage Manager (imsm).  
Now we generate the file that `mdadm` will use to assemble the array on boot.

<pre>
mdadm -I -e imsm /dev/<b>md127</b>
mdadm --examine --scan >> /mnt/etc/mdadm.conf
</pre>

Now, we need to enable `mdadm` uppon boot in order to assemble the RAID array.

<pre>
nano /etc/mkinitcpio.conf  
</pre>

Edit the following like, in the correct position.

<pre>
HOOKS="... block <b>mdadm_udev</b> filesystems ..."
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz  
https://wiki.archlinux.org/index.php/RAID#Installing_Arch_Linux_on_RAID  
https://forums.gentoo.org/viewtopic-t-888520.html
</sup></sub>

###### (Optional) Create initial ramdisk

This was already created when the base system was installed.  
This is only needed if we add any modules, like adding the `mdadm` module.

<pre>
mkinitcpio -p linux
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/mkinitcpio
</sup></sub>

##### Configure GRUB2

<pre>
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

The `grub-mkconfig` command is equal to `update-grub` on Ubuntu, that is just a wrapper.

##### Install GRUB2 to the MBR

This will install the bootloader to the first sector of the disk.  
It is possible to install to a partition, but is not recommended.  
This will be needed if you formated the partition where `/etc/default/grub` or `/etc/grub.d/` was in.

The parameter `--target=i386-pc` instructs grub-install to install for BIOS systems only.  
It is recommended to always use this option to remove ambiguity in grub-install. 

Desktop:  
<pre>
grub-install -target=i386-pc --recheck /dev/<b>sda</b>
</pre>

Notebook:  
<pre>
grub-install --target=i386-pc --recheck /dev/<b>md125</b>
</pre>

##### Change the root password

<pre>
passwd
</pre>

###### (Optional) Install necessary wireless drivers

You will not be able to access `wifi-menu`, without the following packages once we reboot.

<pre>
pacman -S iw wpa_supplicant dialog wpa_actiond
</pre>

##### Exit change-root

<pre>
exit
</pre>

##### Reboot

Now we can reboot and remove the USB drive.

<pre>
systemctl reboot
</pre>

<sub><sup>
General References:  
[Video - System Installation](https://www.youtube.com/watch?v=kQFzVG4wZEg)  
[Video - From Post-Install to Xorg](https://www.youtube.com/watch?v=DAmXKDJ3D7M)  
[Video - Using AUI](https://www.youtube.com/watch?v=TLh44czUea0) 
[Install script AUI](https://github.com/helmuthdu/aui)
</sup></sub>
