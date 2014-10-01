# ArchLinux Instalation notes.

I created these notes for my desktop and notebook installations.  
The desktop uses a single SSD for the OS, so it will use some recommend SSD optimizations.  
The notebook (Asus UX51VZ) uses two SSD's on a RAID 0 configuration with dualboot with Windows 7.

### <font color='red'>Create Bootable USB</font>

##### Windows:

* This will irrevocably destroy all data. 

> Download: [http://sourceforge.net/projects/usbwriter/](http://sourceforge.net/projects/usbwriter/)

<sub><sup>
References:  
[https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#Using_USBwriter](https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#Using_USBwriter)  
</sup></sub>

##### Linux:

* This will irrevocably destroy all data on /dev/sdx.
* Replace sdX with yout drive, do not append partition number
* Find out the name of your USB drive with `lsblk`. Make sure that it is not mounted.

> dd bs=4M if=/path/to/archlinux.iso of=/dev/**sdX** && sync

<sub><sup>
References:  
[https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#In_GNU.2FLinux](https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#In_GNU.2FLinux)  
</sup></sub>

### <font color='red'>Restore partitions on the USB back to normal</font>

##### Windows:

* This will irrevocably destroy all data.  

> Open an elevated command prompt.  
Run diskpart  
List disk  
Find the disk number of the USB drive (it should be obvious going by the size)  
Select disk X  
List partition  
Select partition X  
Delete partition  
Create partition primary  
Exit

<sub><sup>
References:  
[http://superuser.com/questions/536813/how-to-delete-a-partition-on-a-usb-drive](http://superuser.com/questions/536813/how-to-delete-a-partition-on-a-usb-drive)  
</sup></sub>

##### Linux:

* This will irrevocably destroy all data on /dev/sdx.
* First we need to delete the old partitions that remain on the USB key:
> Open a terminal and type sudo su  
Type fdisk -l and note your USB drive letter.  
Type fdisk /dev/sdx (replacing x with your drive letter)  
Type d to proceed to delete a partition  
Type 1 to select the 1st partition and press enter  
Type d to proceed to delete another partition (fdisk should automatically select the second partition)

* Next we need to create the new partition:  
> Type n to make a new partition  
Type p to make this partition primary and press enter  
Type 1 to make this the first partition and then press enter  
Press enter to accept the default first cylinder  
Press enter again to accept the default last cylinder  
Type w to write the new partition information to the USB key  
Type umount /dev/sdx (replacing x with your drive letter)  

* The last step is to create the fat filesystem:  
> Type mkfs.vfat -F 32 /dev/sdx1 (replacing x with your USB key drive letter)

<sub><sup>
References:  
[http://dottheslash.wordpress.com/2011/11/29/deleting-all-partitions-on-a-usb-drive/](http://dottheslash.wordpress.com/2011/11/29/deleting-all-partitions-on-a-usb-drive/)  
</sup></sub>

### <font color='red'>Install base system</font>

##### Check avalable keyboard layouts:

> ls /usr/share/kbd/keymaps/i386/qwerty

##### Load keyboard layout:

> loadkeys pt-latin9

###### (Optional) Update mirrorlist:

> Get mirrors:  [https://www.archlinux.org/mirrorlist/ ](https://www.archlinux.org/mirrorlist/)   
> nano /etc/pacman.d/mirrorlist

###### (Optional) (RAID) Create file to assembly RAID arrays:

* This is required, for the RAID 0 configuration.
* mdadm will use the information generated here to assemble the array on boot.
* So, later we will have to enable the mdadm module itself, in order to load these configs.  
* Also, even if the wiki says that that mdraid is used for fake raid systems, Intel advises the use of mdadm for their boards.

> mdadm -I -e imsm /dev/md127  
> mdadm --examine --scan >> /mnt/etc/mdadm.conf

<sub><sup>
References:  
[https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz](https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz)  
[https://wiki.archlinux.org/index.php/RAID#Installing_Arch_Linux_on_RAID](https://wiki.archlinux.org/index.php/RAID#Installing_Arch_Linux_on_RAID)  
[https://forums.gentoo.org/viewtopic-t-888520.html](https://forums.gentoo.org/viewtopic-t-888520.html)  
</sup></sub>

###### (Optional) Partition plan:

* Before any change we should get to know the details of each recommended partition.

> [https://wiki.archlinux.org/index.php/partitioning#Partition_scheme](https://wiki.archlinux.org/index.php/partitioning#Partition_scheme)  
[http://en.wikipedia.org/wiki/Disk_partitioning#Benefits_of_multiple_partitions](http://en.wikipedia.org/wiki/Disk_partitioning#Benefits_of_multiple_partitions)

##### Check existing partitions:  

> fdisk -l

##### Change MBR partitions:

> cfdisk /dev/sd**X**

##### Change GPT partitions:

> cgdisk /dev/sd**X**

###### (Optional) (RAID) Find out chunk and block size:

* This is needed to correctly format the raid array.
* Use the following command and find out the chunk size (128k in my case).
* The blocksize is normally 4k, it is used for somewhat large files.

> mdadm -E /dev/md127  

* Stride = (chunk size/block size). 
 * Stride = 128 / 4 = 32
* Stripe-width = (number of physical data disks * stride). 
 * Stripe-width = 2 * 32 = 64

##### Format Filesystem:

* Dektop:  
> mkfs.ext4 -v -L Arch -E discard /dev/sda1

* Notebook:
> mkfs.ext4 -v -L Arch -b 4096 -E stride=32,stripe-width=64,discard /dev/md125p4

<sub><sup>
References:  
[https://wiki.debian.org/SSDOptimization#Mounting_SSD_filesystems](https://wiki.debian.org/SSDOptimization#Mounting_SSD_filesystems)  
[http://blog.nuclex-games.com/2009/12/aligning-an-ssd-on-linux/](http://blog.nuclex-games.com/2009/12/aligning-an-ssd-on-linux/)  
</sup></sub>

##### Create folders and mount partitions:

* Dektop:  
> mount /dev/sda1 /mnt

* Notebook:
> mount /dev/md125p4 /mnt

* (Optional) Folders:
> mkdir /mnt/boot /mnt/home /mnt/var  
mount /dev/sdaX /mnt/boot  
mount /dev/sdaX /mnt/home  
mount /dev/sdaX /mnt/var  

##### SWAP

A SWAP is used for moving data in memory to disk and to enable the hibernate functionality.  
In this case, I will be using a swap file instead of a partition because it is easer to resize and having a dedicated partition for this does not seem useful.  
Note that the file should have at least the same size of the physical RAM, although for me, I do not plan on hibernate with  my RAM full, so I will give less space.

###### Create SWAP file
> fallocate -l 6G /mnt/swapfile  
chmod 600 /mnt/swapfile  
mkswap /mnt/swapfile  
swapon /mnt/swapfile  


###### Create SWAP partition

> mkswap /mnt/sda1	
swapon /mnt/sda1

###### Check SWAP status

> swapon -s

<sub><sup>
References:  
[https://wiki.archlinux.org/index.php/Swap#Swap_file](https://wiki.archlinux.org/index.php/Swap#Swap_file)  
</sup></sub>




<sub><sup>
References:  
[Video - System Installation ](https://www.youtube.com/watch?v=kQFzVG4wZEg)  
[Video - From Post-Install to Xorg](https://www.youtube.com/watch?v=DAmXKDJ3D7M)  
[Video - Using AUI](https://www.youtube.com/watch?v=TLh44czUea0) 
</sup></sub>
