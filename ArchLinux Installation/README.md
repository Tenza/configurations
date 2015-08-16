# ArchLinux Instalation Notes

These notes were created for my desktop and notebook installations.  
The desktop uses a single SSD and the notebook (Asus UX51VZ) uses two SSD's on a RAID 0 configuration.  
Both have dualboot with Windows 7, and since both use SSD drives we will make some recommend optimizations.

Some configurations in here might be outdated, always check the archwiki.

<sub><sup>
References:  
[Video - System Installation](https://www.youtube.com/watch?v=kQFzVG4wZEg)  
[Video - From Post-Install to Xorg](https://www.youtube.com/watch?v=DAmXKDJ3D7M)  
[Video - Using AUI](https://www.youtube.com/watch?v=TLh44czUea0) 
[Install script AUI](https://github.com/helmuthdu/aui)
</sup></sub>

### Create bootable USB

##### Windows:

This will irrevocably destroy all data.  

> Download: http://sourceforge.net/projects/usbwriter/

<sub><sup>
References:
https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#Using_USBwriter
</sup></sub>

##### Linux:

This will irrevocably destroy all data on /dev/sdX.  
Find out the name of the USB drive with `lsblk`.  
Do not append partition a number and make sure the drive is not mounted.  

> dd bs=4M if=/path/to/archlinux.iso of=/dev/**sdX** && sync

<sub><sup>
References:
https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media#In_GNU.2FLinux
</sup></sub>

### Restore partitions on the USB

##### Windows:

This will irrevocably destroy all data.  

> Open an elevated command prompt.  
Type `diskpart`  
Type `List disk`  
Find the disk number of the USB drive (it should be obvious going by the size)  
Type `Select disk X`  
Type `List partition`  
Type `Select partition X`  
Type `Delete partition`  
Type `Create partition primary`  
Type `Exit`

<sub><sup>
References:
http://superuser.com/questions/536813/how-to-delete-a-partition-on-a-usb-drive
</sup></sub>

##### Linux:

This will irrevocably destroy all data on /dev/sdX.  
First we need to delete the old partitions that remain on the USB key:  

> Open a terminal and type sudo su  
Type `fdisk -l` and note your USB drive letter.  
Type `fdisk /dev/sdX`  
Type `d` to proceed to delete a partition  
Type `1` to select the 1st partition and press enter  
Type `d` to proceed to delete another partition (fdisk should automatically select the second partition)

Next we need to create the new partition:  

> Type `n` to make a new partition  
Type `p` to make this partition primary and press enter  
Type `1` to make this the first partition and then press enter  
Press enter to accept the default first cylinder  
Press enter again to accept the default last cylinder  
Type `w` to write the new partition information to the USB key  
Type `umount /dev/sdX`

The last step is to create the fat filesystem:  

> Type `mkfs.vfat -F 32 /dev/sdX1`

<sub><sup>
References:
http://dottheslash.wordpress.com/2011/11/29/deleting-all-partitions-on-a-usb-drive/
</sup></sub>

### Install base system

##### Check avalable keyboard layouts:

> ls /usr/share/kbd/keymaps/i386/qwerty

##### Load keyboard layout:

> loadkeys pt-latin9

###### (Optional) Update mirrorlist:

> Get mirrors: https://www.archlinux.org/mirrorlist/  
nano /etc/pacman.d/mirrorlist

See how to optimize bellow.

###### (Optional) (RAID) Create file to assembly RAID arrays:

This is required, for the RAID 0 configuration.  
`mdadm` will use the information generated here to assemble the array on boot.  
So, later we will have to enable the `mdadm` module itself, in order to load these configurations.  
Also, note that even if the wiki says that `mdraid` is used for fake raid systems, Intel advises the use of `mdadm` for their boards.  
Find out where your RAID array is located with:

> mdadm --detail-platform  
mdadm -E /dev/**mdX**

In my case the array is located on md127 and uses Intel Matrix Storage Manager (imsm).  

> mdadm -I -e imsm /dev/**md127**  
mdadm --examine --scan >> /mnt/etc/mdadm.conf

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz  
https://wiki.archlinux.org/index.php/RAID#Installing_Arch_Linux_on_RAID  
https://forums.gentoo.org/viewtopic-t-888520.html
</sup></sub>

###### (Optional) Partition plan:

Before any change we should get to know the details of each recommended partition.

> https://wiki.archlinux.org/index.php/partitioning#Partition_scheme  
http://en.wikipedia.org/wiki/Disk_partitioning#Benefits_of_multiple_partitions

##### Check existing partitions:  

> fdisk -l

##### (1) Change MBR partitions:

> cfdisk /dev/**sdX**

##### (2) Change GPT partitions:

> cgdisk /dev/**sdX**

###### (Optional) (RAID) Find out chunk and block size:

This is needed to correctly format the raid array.  
Use the following command and find out the chunk size (128k in my case).  
The blocksize is normally 4k, it is used for somewhat large files.  

> mdadm -E /dev/**md127**  

* Stride = (chunk size/block size). 
 * Stride = 128 / 4 = 32
* Stripe-width = (number of physical data disks * stride). 
 * Stripe-width = 2 * 32 = 64

##### Format the filesystem with SSD and RAID optimizations:

Dektop:  
> mkfs.ext4 -v -L Arch -E discard /dev/**sda1**

Notebook:  
> mkfs.ext4 -v -L Arch -b 4096 -E stride=32,stripe-width=64,discard /dev/**md125p4**

<sub><sup>
References:  
https://wiki.debian.org/SSDOptimization#Mounting_SSD_filesystems  
http://blog.nuclex-games.com/2009/12/aligning-an-ssd-on-linux/  
</sup></sub>

##### Create folders and mount partitions:

Dektop:  
> mount /dev/**sda1** /mnt

Notebook:  
> mount /dev/**md125p4** /mnt

(Optional) Folders for other partitions:  

> mkdir /mnt/boot /mnt/home /mnt/var  
mount /dev/**sdaX** /mnt/boot  
mount /dev/**sdaX** /mnt/home  
mount /dev/**sdaX** /mnt/var  

##### SWAP

A SWAP is used for moving data in memory to disk and to enable the hibernate functionality.  
In this case, I will be using a swap file instead of a partition because it is easer to resize and having a dedicated partition for this does not seem useful.  
Note that the file should have at least the same size of the physical RAM, although for me, I do not plan on hibernate with my RAM full, so I will give less space.

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Swap#Swap_file
</sup></sub>

##### (1) Create SWAP file

> fallocate -l 6G /mnt/swapfile  
chmod 600 /mnt/swapfile  
mkswap /mnt/swapfile  
swapon /mnt/swapfile  

##### (2) Create SWAP partition

> mkswap /mnt/**sda1**	
swapon /mnt/**sda1**

##### Check SWAP status

> swapon -s

##### Check network

Before we start with the installation of the base system, we need to make sure that the network is working.
If we are using a wired connection, DHCP should be activated by default.

> ping -c 3 google.com

###### (Optional) Activate wireless connectivity

> wifi-menu

##### Install base system

> pacstrap -i /mnt base base-devel

##### Install bootloader

The bootloader to be installed depends on the partition style and the board firmware type.

Example 1, if we have a board with UEFI, and Windows 7 installed, we can assume that the CMS mode is enabled and the partition style is MBR. Knowing this, we should use a bootloader for a BIOS system.

Example 2, if we have a board with UEFI, and Windows 8 installed, we can assume that the CMS mode is disabled and the partition style is GPT. Knowing this, we should use a bootloader for a UEFI system.

https://wiki.archlinux.org/index.php/Windows_and_Arch_Dual_Boot#Bootloader_UEFI_vs_BIOS_limitations

---

The NT bootloader is located on a 100MB "System Reserved" partition. This cannot be erased even with a diferent bootloader because other bootloaders cannot directly start the OS. They have to chainload with the NT bootloader in order to start Windows.

---

Using `pacstrap` or `pacman -S grub` is the same thing. But the former will get the correct package if the name changes.

##### Install GRUB2 for BIOS-MBR system

> pacstrap -i /mnt grub-bios 

##### Generate filesystem table

> genfstab -p /mnt >> /mnt/etc/fstab

<sub><sup>
References:
http://unix.stackexchange.com/questions/124733/what-does-genfstabs-p-option-do
</sup></sub>

###### (Optional) Change filesystem table flags for SSD optimization.

> nano /mnt/etc/fstab  
Add to the SSD disk parameters: `discard` to activate TRIM, if you are sure that the SSD has support.  
Update the SSD disk parameters: `realatime` for `noatime` it eliminates the need by the system to make writes to the file system for files which are simply being read.  
Make sure the SWAP is not on `/mnt` and rather on `/`  

Checks:
> If the kernel knows about SSDs: /sys/block/sd?/queue/rotational  
Check for TRIM support: hdparm -I /dev/sd? | grep TRIM  

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Solid_State_Drives#noatime_Mount_Flag  
https://wiki.archlinux.org/index.php/Solid_State_Drives#Enable_TRIM_by_Mount_Flags
</sup></sub>

##### Enter change-root

Once we change root we will be inside our system.  
From this moment, We no longer refer to the `/mnt` mount point .

> arch-chroot /mnt

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Change_root
</sup></sub>

###### (Optional) (RAID) Activate RAID module

As said before, we need to enable `mdadm` uppon boot in order to assemble the RAID arrays.

> nano /etc/mkinitcpio.conf  
HOOKS="... block **mdadm_udev** filesystems ..."

###### (Optional) Create initial ramdisk

This was already created when the base system was intalled.  
This is only needed if we add any modules, like the above step.

> mkinitcpio -p linux

<sub><sup>
References:
https://wiki.archlinux.org/index.php/mkinitcpio
</sup></sub>

##### Configure GRUB2

> grub-mkconfig -o /boot/grub/grub.cfg

The `grub-mkconfig` command is equal to `update-grub` on Ubuntu, that is just a wrapper.

##### Install GRUB2 to the MBR

This will install the bootloader to the first sector of the disk.  
It is possible to install to a partition, but is not recommended.

`--target=i386-pc` instructs grub-install to install for BIOS systems only.  
It is recommended to always use this option to remove ambiguity in grub-install. 

Desktop:  
> grub-install -target=i386-pc --recheck /dev/**sda**

Notebook:  
> grub-install --target=i386-pc --recheck /dev/**md125**

##### Change the root password

> passwd

###### (Optional) Install necessary wireless drivers

You will not be able to access `wifi-menu`, without there packages once we reboot.
> pacman -S iw wpa_supplicant dialog wpa_actiond

##### Exit change-root

> exit

##### Reboot

Now we can reboot and remove the USB drive.

> systemctl reboot

### Configure the system

##### Configure persistent keymap

> localectl set-keymap pt-latin9  
localectl set-x11-keymap pt  

> [filipe@filipe-desktop ~]$ localectl status  
System Locale: LANG=pt_PT.UTF-8  
VC Keymap: pt-latin9  
X11 Layout: pt

##### Configure hostname

This is the name that will showup in the console, por example:
filipe@filipe-desktop

> hostnamectl set-hostname filipe-desktop

##### Configure timezone 

Get time zone info:  
> ls /usr/share/zoneinfo

In my case is Portugal, so:
> timedatectl set-timezone Portugal

##### Configure locale

> nano /etc/locale.gen

Remove the comment from the lines with the country code. pt_PT are 3.

> locale-gen  
localectl set-locale LANG="pt_PT-UTF-8"

##### Configure network cards

There are multiple possible configurations. The following is the best for me.  
https://wiki.archlinux.org/index.php/Beginners%27_guide#Configure_the_network

Get interfaces name:
> ip link

In my case the interfaces needed are:  
Wired: enp2s0  
Wireless: wlp3s0

##### Configure wired connection with DHCP

> systemctl start dhcpcd@enp2s0.service

##### Configure wireless connection with wifi-menu

> wifi-menu

###### (Optional) Load on boot

This will activate the connections on boot, but keep in mind, that these will have to be stopped and disabled when we install the `networkmanager` package once we have a graphical enviroment.

> systemctl enable dhcpcd@enp2s0.service  
systemctl enable netctl-auto@wlp3s0.service

##### (1) Configure hardware clock for UTC

By default linux uses UTC, so we should instal NTP to sync the time online.

> pacman -S ntp  
systemctl enable ndpd.service

###### (2) Configure hardware clock for Localtime

The only reason to ever do this, is if we are in a dualboot configuration and there is already an OS managing time and DST switching while storing the RTC time in localtime, such as Windows. But even so, it would be preferable to force Windows to use UTC, rather than forcing Linux to localtime. To do this, just install the Meinberg NTP Software.

With this said, keep in mind that we still have to logon in Windows at least two times a year (in Spring and Autumn) when DST kicks i.n

> timedatectl set-local-rtc true

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Time  
http://www.satsignal.eu/ntp/setup.html  
http://www.meinbergglobal.com/english/sw/ntp.htm
</sup></sub>

#### Dualboot

First we install the package responsible for detecting other OS instalations:

> pacman -S os-prober

<sub><sup>
References:
https://wiki.archlinux.org/index.php/GRUB#Dual-booting
</sup></sub>

##### Windows already instaled:

If we are installing Arch and Windows is already on the machine, assuming the bootloader was already installed to the MBR, we simply need to refresh the config file.

> grub-mkconfig -o /boot/grub/grub.cfg

The os-prober will be automatically called by `grub-mkconfig` and will add an entry to Windows. This has worked for me even on a dualboot with a RAID array.

But if for some reason it faild we can try and add a costum entry:

> nano /etc/grub.d/40_custom

```
menuentry{
	echo “Loading Windows 7”
	insmod part_msdos
	insmod ntfs
	search --fs-uuid --no-floppy -set=root XXXXXXXXXXXXXX
	chainloader +1
}
```

Use `blkid` to get the uuid of the disk and replace the 'XXX'. And re-run the command:

> grub-mkconfig -o /boot/grub/grub.cfg

##### Arch already instaled:

If we are installing Windows and Arch is already on the machine, we have to reinstall the bootloader because Windows automatically overwrites the MBR. To do so, we have to start Arch Live and do: 

> loadkeys pt-latin9  
mount /dev/**sda** /mnt  
arch-chroot /mnt  
cfdisk /dev/sdX  (if you need to set the bootflag)  
grub-mkconfig -o /boot/grub/grub.cfg  
grub-install -target=i386-pc --recheck /dev/**sda**

If you need any extra package aditionaly use:
> wifi-menu  
pacman -Syy  
pacman -S os-prober  

##### Custom Entries

> nano /etc/grub.d/40_custom

```
menuentry "System restart" {
	echo "System rebooting..."
	reboot
}

menuentry "System shutdown" {
	echo "System shutting down..."
	halt
}
```

> grub-mkconfig -o /boot/grub/grub.cfg

We can also change the text displayed on the bootloader if we manually edit the file.
Keep in mind that these changes will be lost when we rebuild the file. Do this with extreme caution.
If we delete something needed, the machine might not boot (but we  can always fix it with a live cd :P).
Also, to use skins, see the next part.

> nano /boot/grub/grub.cfg

##### Reboot

> systemctl reboot

### Post-Install to XOrg

##### Add new user

It is good practice to use a normal user and elevate to root only when necessary.
Also, decide if you want to create a new primary group or add to the existing `users` group

> useradd -m -G wheel -s /bin/bash filipe OR useradd -m -g users -G wheel -s /bin/bash filipe
chfn filipe  
passwd filipe

##### Give sudo permissons to new user

Trick visudo to open with nano:
> EDITOR=nano visudo

Add user to the section “User privilegie specification”:
> Filipe ALL=(ALL) ALL

Logout from root and enter the new account:
> logout

##### Activate multilib repo

Remove comments from [multilib]:
> sudo nano /etc/pacman.conf  
sudo pacman -Syy

##### Install Packer

> sudo pacman -S wget git jshon expac  
wget https://aur.archlinux.org/packages/pa/packer/packer.tar.gz  
tar zxvf packer.tar.gz  
cd packer && makepkg  
sudo pacman -U packer (press tab)

Once installed, cleanup:

> rm -R packer  
rm packer.tar.gz

<sub><sup>
References:
http://www.cyberciti.biz/faq/unpack-tgz-linux-command-line/
</sup></sub>

##### Sound drivers

ALSA is already apart of the Kernel, but the channels are muted by default.

> sudo pacman -S alsa-utils  
run `alsamixer`  
Press H to unmute. Press F1 for help.  
run `speaker-test`

Install the plugins for high quality resampling:
> sudo pacman -S alsa-plugins

Now lets install the pulseaudio server:
> sudo pacman -S pulseaudio pulseaudio-alsa 

If we are in a x86_64 system we need to have sound for 32-bit multilib programs like Wine, Skype and Steam: 
> sudo pacman -S lib32-libpulse lib32-alsa-plugins

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Advanced_Linux_Sound_Architecture#High_quality_resampling  
https://wiki.archlinux.org/index.php/PulseAudio#Back-end_configuration
</sup></sub>

##### XOrg

> sudo pacman -S xorg-server xorg-server-utils xorg-xinit mesa

<sub><sup>
References:
https://wiki.archlinux.org/index.php/xorg#Installation
</sup></sub>

##### Graphic drivers

Get graphics card:
> lspci | grep VGA

Desktop:

The graphics card is too old, and there is no support for it, so I use the open-source drivers.

Find open-source driver:
> sudo pacman -Ss xf86-video | less

Install:
> sudo pacman -S mesa-dri xf86-video-vesa xf86-video-ati

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/ATI#Installation
https://wiki.archlinux.org/index.php/Xorg#Driver_installation
</sup></sub>

Notebook:

This notebook has two graphics cards, intel and nvidia.
In order to manage them we use `bumblebee` that is Optimus for GNU/Linux.

> sudo pacman -S mesa-dri xf86-video-intel nvidia bumblebee bbswitch

Configure bumblebee:

> sudo gpasswd -a filipe bumblebee  
sudo systemctl enable bumblebeed.service 

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Bumblebee#Installing_Bumblebee_with_Intel.2FNVIDIA  
https://wiki.archlinux.org/index.php/Bumblebee#Start_Bumblebee
https://wiki.archlinux.org/index.php/Hybrid_graphics#ATI_Dynamic_Switchable_Graphics
</sup></sub>

##### Input drivers

Desktop and Notebook:
> sudo pacman -S xf86-input-mouse xf86-input-keyboard

Notebook:
> sudo pacman -S xf86-input-synaptics 

##### Test default enviroment

Make sure XOrg is working before we install a desktop enviroment.

> sudo pacman -S xorg-twm xorg-xclock xterm  
startx

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Xorg#Manually
</sup></sub>

###### (1) Install KDE 4

> sudo pacman -S kdebase kde-l10n-pt  
sudo systemctl enable kdm.service  
sudo systemctl reboot

For the photon backend, use VLC because it has the best upstream support.  
For the Kactivities, use the new `kactivities-frameworks`instead of `kactivities4`  
For the fonts, use ttf-oxygen because it is KDE.

<sub><sup>
References:
https://wiki.archlinux.org/index.php/KDE
</sup></sub>

###### (2) Install KDE 5

At the moment this is just experimental. There are still some problems regarding the tray area being changed, as well as the look on some Qt4 applications. Even though there is a themes for them.

If KDE4 is installed, and since KDE4 and KDE5 cannot run together we need to remove it first:

> sudo pacman -Rnc kdebase-workspace 

Install KDE 5:

> sudo pacman -S plasma kde-l10n-pt

Instead of KDM it is recommended to use the newer SDDM display manager

> sudo pacman -S sddm

Activate SDDM and create the config file

> systemctl disable kdm && systemctl enable sddm  
> sddm --example-config > /etc/sddm.conf

Finally, apply the newer `breeze` theme to the display manager, because the default one is just ugly

> sudo nano /etc/sddm.conf

And change the theme section according to this:
```
[Theme]
Current=breeze
CursorTheme=breeze_cursors
FacesDir=/usr/share/sddm/faces
ThemeDir=/usr/share/sddm/themes
```

OR you can do it with GUI, just install:

> sudo pacman -S sddm-kcm

Reboot and go to `Setting > Startup and Shutdown > Login Screen. (2nd tab)` and choose the `Breeze` theme.

To make things look better and more uniform install and select the following themes.
Before instalation tt is better to look on the wiki, because at the moment there are no breeze-based themes for GTK, and this might be totally diferent. 

> sudo pacman -S breeze-kde4 qtconfig-qt4 

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Plasma  
https://wiki.archlinux.org/index.php/SDDM  
http://www.linuxveda.com/2015/02/27/how-to-install-kdes-plasma-5-on-arch-linux/
</sup></sub>

### Configure X11 system

##### Fonts

For fonts I like to use the infinality bundle, they improve text rendering and have a package of hand picked fonts.  
First, we need to add their repository and key.
Also, if the terminal within KDE is messed up, press `Ctrl+Alt+F1` to `F6` to change to a virtual console provided by getty. In order to get back to the X session, press `Ctrl+Alt+F7`.

> nano /etc/pacman.conf  

> [infinality-bundle]  
Server = http://bohoomil.com/repo/$arch  
[infinality-bundle-multilib]  
Server = http://bohoomil.com/repo/multilib/$arch  
[infinality-bundle-fonts]  
Server = http://bohoomil.com/repo/fonts

Then we add the key and install everything:

> pacman-key -r 962DDE58  
pacman-key --lsign-key 962DDE58  
pacman -Syyu  
pacman -S infinality-bundle infinality-bundle-multilib ibfonts-meta-base

<sub><sup>
References:  
http://bohoomil.com/  
https://wiki.archlinux.org/index.php/Infinality-bundle%2Bfonts#Installation  
https://wiki.archlinux.org/index.php/Unofficial_user_repositories#infinality-bundle  
https://wiki.archlinux.org/index.php/Unofficial_user_repositories#infinality-bundle-multilib  
https://wiki.archlinux.org/index.php/Unofficial_user_repositories#infinality-bundle-fonts  
https://bbs.archlinux.org/viewtopic.php?id=162098
</sup></sub>

##### MS Fonts

To install Microsoft fonts we will use the Legacy packages, because the new packages require us to have a mounted Windows partition or access to a setup or installation media, and I dont really want to waste time with this BS.

> sudo packer -S ttf-ms-fonts ttf-vista-fonts

<sub><sup>
References:
https://wiki.archlinux.org/index.php/MS_fonts
</sup></sub>

##### Optimize Mirrorlist

> Get all up-to-date mirrors here: https://www.archlinux.org/mirrorlist/all/  
Goto: `cd /etc/pacman.d`  
Backup: `sudo mv mirrorlist mirrorlist.bak`  
Create new mirror list and paste the mirrors from the link: `sudo nano mirrorlist.new`  
Remove commented lines with: `sudo sed -i "s/#Server/Server/g" mirrorlist.new`  
Rank all the mirrors: `sudo rankmirrors -v -n 0 mirrorlist.new`  
Finnaly create and paste the result on the mirrorslist file and delete mirrorlist.new  

<sub><sup>
References:  
http://www.garron.me/en/go2linux/how-to-find-the-fastest-archlinux-mirrors.html  
http://bbs.archbang.org/viewtopic.php?id=3007
</sup></sub>

##### KDE volume control

To have access to the tray applet:

> sudo pacman -S kdemultimedia-kmix

##### KDE network manager

This will install the network tray applet, as well as its dependencies, that include the main package `networkmanager`.

> sudo pacman -S kdeplasma-applets-plasma-nm

If you enabled DHCP or netctl-auto on boot stop, disable them and start the new service:

> sudo systemctl -t service  
sudo systemctl stop dhcpcd@enp2s0.service   
sudo systemctl stop netctl-auto@wlp3s0.service  
sudo systemctl disable dhcpcd@enp2s0.service  
sudo systemctl disable netctl-auto@wlp3s0.service  
sudo systemctl enable NetworkManager.service  
reboot

Once we reboot and we use the networkmanager for a wireless connection, it will open KWallet in order to save the password. More below.

<sub><sup>
References:
https://wiki.archlinux.org/index.php/NetworkManager#KDE
</sup></sub>

##### KDE KWallet

If the KWallet is already open, simply use the default configuration, and create the default `kdewallet` wallet.  
If not, install the following package and open in `System Configurations > Account Details > Wallet` or `ALT+F2` and type `kwalletmanager`. 

> sudo pacman -S kdeutils-kwalletmanager  

Now, if Kwallet is prompting you at each startup for the password, probably due to the saved wireless password, you can simply change its password to blank.

<sub><sup>
References:  
http://mschlander.wordpress.com/2012/08/01/kwallet-is-annoying/   http://unix.stackexchange.com/questions/34186/automatically-opening-kwallet-while-logging-in-to-kde  
</sup></sub>

##### Bluetooth

This will install the bluetooth tray applet, as well as its dependencies, that include the main package `bluez`.

> sudo pacman -S bluedevil

Activate the bluetooth card, but be aware that some Bluetooth adapters are bundled with a Wi-Fi card (e.g. Intel Centrino). These require that the Wi-Fi card is first enabled (typically a keyboard shortcut on a laptop) in order to make the Bluetooth adapter visible to the kernel.

> systemctl start bluetooth  
systemctl enable bluetooth

There are some known problems when the bluetooth deamon starts, but fortunatly bluetooth still works.  
To see the errors do:

> systemctl status bluetooth

> Sap driver initialization failed.  
sap-server: Operation not permitted (1)  
hci0 Load Connection Parameters failed: Unknown Command (0x01)

https://bugs.archlinux.org/task/41247

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Bluetooth#BlueDevil  
https://wiki.archlinux.org/index.php/Bluetooth#Installation  
</sup></sub>

##### CD/DVD/Bluray Drive

Arch doesn't mount the drive unless its necessary (i.e. a DVD or CD is loaded into the drive).  
The mount command will only work if the drive is already mounted, but it will probably not be needed because it should be auto-mounted once the media is loaded.

To burn and rip CD's and DVD's I like the K3b that is part of the KDE suite.

> sudo pacman -S k3b

##### Uniform Look

To make GTK based applications look like Qt based, I prefer `oxygen-gtk` instead of `qtcurve`.

> sudo pacman -S oxygen-gtk2 kde-gtk-config-kde4  
> packer -S oxygen-gtk3-git

Go to, and change to `oxygen-gtk`:  
System Settings > Application Appearance > GTK

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Uniform_Look_for_Qt_and_GTK_Applications  
http://askubuntu.com/questions/50928/qtcurve-vs-oxygen-gtk-theme
</sup></sub>

##### Mount on boot

> sudo pacman -S ntfs-3g  
sudo nano /etc/fstab

*With dmask and fmask flags.*

Desktop:

Besides mounting on boot, we will set access permissions to only the created user.  
I do this because I have some untrusted programs running on locked accounts, i.e Skype.  
But this apporach has a big downside, we will not be able to change permissions on the fly.

> /dev/sdb1 /media/Dados1 ntfs defaults,uid=1000,dmask=027,fmask=137 0 0  
/dev/sdc1 /media/Dados2 ntfs defaults,uid=1000,dmask=027,fmask=137 0 0  
/dev/sdd /media/Dados3 ntfs defaults,uid=1000,dmask=027,fmask=137 0 0  

Notebook:

> /dev/mp125p3 /media/Dados ntfs defaults,noatime,discard,uid=1000,dmask=027,fmask=137 0 0

*With permission flag.*

To be able to change permissions, use the following flag.  
ntfs and ntfs-3g are the same, check with: `ls /sbin/mount.ntfs* -l`

> /dev/sdb1               /media/Dados1   ntfs-3g         defaults,permissions     0 0  
/dev/sdc1               /media/Dados2   ntfs-3g         defaults,permissions     0 0  
/dev/sdd                /media/Dados3   ntfs-3g         defaults,permissions     0 0  

Then, set the files permissions and ownership with:

> sudo chown -R filipe:users /media/Dados3/  
find /media/Dados3/ -type d -exec chmod 750 {} \;  
find /media/Dados3/ -type f -exec chmod 640 {} \;  
Adjust commands to all other storage drives.  

<sub><sup>
References:  
http://en.wikipedia.org/wiki/Fmask#Example  
https://wiki.archlinux.org/index.php/NTFS-3G  
http://www.omaroid.com/fstab-permission-masks-explained/  
http://askubuntu.com/questions/429848/dmask-and-fmask-mount-options  
http://askubuntu.com/questions/113733/how-do-i-correctly-mount-a-ntfs-partition-in-etc-fstab  
http://serverfault.com/questions/304354/fstab-filesystem-type-for-ntfs-ntfs-or-ntfs-3g  
http://man7.org/linux/man-pages/man8/mount.8.html#FILESYSTEM-INDEPENDENT_MOUNT%20OPTIONS  
</sup></sub>

##### Change Kernel I/O scheduler

When using only SSD's for the OS, one setting that can boost your storage performance dramatically is changing the I/O scheduler.

> sudo nano /etc/default/grub  
Change the line:  
GRUB_CMDLINE_LINUX_DEFAULT="quiet"  
To:  
GRUB_CMDLINE_LINUX_DEFAULT="noquiet nosplash elevator=noop"  
sudo grub-mkconfig -o /boot/grub/grub.cfg  

The quiet and splash parameters is just to show the systemd startup and shutdown code. The elevator is the kernel paremeter that changes the I/O scheduler.

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Solid_State_Drives#I.2FO_Scheduler  
http://blog.nashcom.de/nashcomblog.nsf/dx/linux-io-performance-tweek.htm?opendocument&comments  
https://wiki.archlinux.org/index.php/Solid_State_Drives#Kernel_parameter_.28for_a_single_device.29  
</sup></sub>

##### SSD Alignment

What happend:  
I had to format my Windows instalation and I decided to arrange and resize all the partitions.  
Previously I had:  
Windows 230GB, Data 180GB, Linux 50GB  
But I wanted:  
Windows 130GB, Linux 130GB, Data 200GB  

First I backed up my Linux installation to an external storage using EaseUS Partition Master. Then I resized the partitions, and reinstaled windows. Finnaly I put my Linux instalation in place and manually fixed the bootloader with ArchLive. Everything is fine until I ran:

```
$ sudo fdisk -l
...
Partition 3 does not start on physical sector boundary.
Partition 4 does not start on physical sector boundary.
...
```

Then I realized it was an alignment problem, and checked it with:

```
$ sudo parted /dev/md125
GNU Parted 3.2
Using /dev/md125
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) align-check opt 1                                                
1 aligned
(parted) align-check opt 2
2 aligned
(parted) align-check opt 3
3 not aligned
(parted) align-check opt 4
4 not aligned
```

I tried to fix the problem using the following softwares: Paragon-Aligment, AOMEI Partition Assistant and MiniTool, unfortunatly all these software are able to do is align NTFS partitions, and so they only were able to align my Partition 4.

To align an Ext partitions, I did it manually with the help of GParted LiveCD following a procedure recommended by Intel on the following PDF: http://www.intel.com/content/dam/www/public/us/en/documents/technology-briefs/ssd-partition-alignment-tech-brief.pdf

```
Re-align Partitions Without Losing Data

If a partition is misaligned and has data that needs to be preserved, using the open source tool GPARTED is recommended. 
GPARTED can be downloaded from gparted.org. The live image is the best to use for this purpose. 
Procedure for GPARTED:

• Before proceeding – BACK UP THE ENTIRE DRIVE – all partitions should be backed up or cloned for DR purposes
• Boot from the GPARTED Live CD or USB stick
• In the gparted GUI, select your SSD in the upper right drop-down menu
• Click on the first partition
• Click on Resize/Move
• Change Free Space Proceeding to 2MB
• Uncheck Round to Cylinders, or select Align to: MiB
• Click on Resize/Move
• Click OK on the warning page
• Click Apply
• Click Apply in the warning window
• This process can take a great deal of time, depending on the amount of data
• When the move is complete, close out of GPARTED and reboot the system
```

To boot with GParted Live the easiest way is to use `tuxboot`, this tool also recommended by GParted, it downloads and burns GParted live on a USB drive.

> sudo packer -S tuxboot

##### Improve battery life

To improve overall battery life of my notebook I use the package laptop-mode-tools and activate multiple kernel modules, check the references for the description of each activated module.

Laptop tools:
> sudo pacman -S acpid wireless_tools ethtool bluez-utils  
sudo packer -S laptop-mode-tools  
sudo systemctl enable laptop-mode  

For more optimizations install the `powertop` tool by intel.
> sudo pacman -S powertop  
sudo powertop --html=powerreport.html

Now check your $home for the `powerreport.html` file, open it and review the `Tuning` tab.  
It includes the commands necessary for more battery optimizations.

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Laptop_Mode_Tools  
https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz#Powersave_management  
https://wiki.archlinux.org/index.php/powertop  
http://lumbercoder.com/2014/03/02/improve-battery-life-arch-linux.html
</sup></sub>

Kernel Modules:
> sudo nano /etc/default/grub  
GRUB_CMDLINE_LINUX_DEFAULT="noquiet nosplash elevator=noop i915.i915_enable_rc6=1 i915.i915_enable_fbc=1 i915.lvds_downclock=1 pcie_aspm=force drm.vblankoffdelay=1 i915.semaphores=1"  
sudo grub-mkconfig -o /boot/grub/grub.cfg  

<sub><sup>
References:  
https://wiki.ubuntu.com/Kernel/PowerManagement/PowerSavingTweaks  
https://wiki.debian.org/InstallingDebianOn/Asus/UX31a  
https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz#Powersave_management  
https://wiki.archlinux.org/index.php/ASUS_Zenbook_Prime_UX31A#Kernel_Parameters  
https://01.org/linuxgraphics/downloads/2012/intel-2011q4-graphics-stack-release  
</sup></sub>

##### UVC Webcams

To enable external webcam support, activate the following kernel module:
> sudo nano /etc/default/grub  
GRUB_CMDLINE_LINUX_DEFAULT="noquiet nosplash elevator=noop i915.i915_enable_rc6=1 i915.i915_enable_fbc=1 i915.lvds_downclock=1 pcie_aspm=force drm.vblankoffdelay=1 i915.semaphores=1 uvcvideo"  
sudo grub-mkconfig -o /boot/grub/grub.cfg  

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/webcam_setup#linux-uvc
</sup></sub>

##### Bootloader theme

I use the following bootloader theme:
https://github.com/Generator/Grub2-themes

Check the page before following these instructions:
> git clone git://github.com/Generator/Grub2-themes.git # cp -r Grub2-themes/{Archlinux,Archxion} /boot/grub/themes/  
> sudo nano /etc/default/grub  
Change: #GRUB_THEME="/path/to/gfxtheme"   
To: GRUB_THEME="/boot/grub/themes/Archxion/theme.txt"   
Or: GRUB_THEME="/boot/grub/themes/Archlinux/theme.txt"  

> To center the theme, change:  
Change: GRUB_GFXMODE=auto  
To: GRUB_GFXMODE=1024x768  

> Apply changes:  
sudo grub-mkconfig -o /boot/grub/grub.cfg  

##### Samba

Install the Samba and the KDE samba utility:
> sudo pacman -S samba kdenetwork-filesharing   
sudo systemctl enable smbd.socket nmbd  
sudo systemctl start smbd.socket nmbd   

Apply the default settings:
> sudo cp /etc/samba/smb.conf.default /etc/samba/smb.conf

Create a public folder (there are alot of examples on the configuration file):  
And comment out the [homes] and [printers] default shares.
>  [Transfer]  
   comment = Network Transfer  
   path = /media/Dados/Transferências Rede/  
   valid users = filipe  
   public = no  
   writable = yes  
   printable = no  
  
Check if there are no configuration errors:
> testparm

Create a new samba user (this will also create the users file):
> sudo smbpasswd -a filipe

At this point we can test on the localhost with `\\hostname`, to enable network usage, we have to open the following port on the firewall:
> TCP/445 - Samba over TCP  
> UDP/137 - NetBIOS Name Service 

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/samba  
https://wiki.archlinux.org/index.php/Samba/Tips_and_tricks
</sup></sub> 

##### VLC with samba

> Open VLC  
Go to Tools > Preferences  
Under the header Show Settings (bottom left), select the All button  
Select Input / Codecs > Access Modules > SMB on the left  
Input the samba user and password, defined above and leave the Domain empty  
To view log errors, just open "vlc" with the console.

<sub><sup>
References: 
http://jorisvandijk.com/2013/12/24/vlc-wont-play-smb-shares/
</sup></sub> 

##### CUPS

Install the CUPS (deamon) and the KDE CUPS utility:
> sudo pacman -S cups kdeutils-print-manager  
sudo systemctl enable org.cups.cupsd.service  
sudo systemctl start org.cups.cupsd.service  

Install EPSON printer drivers:
> sudo packer -S epson-inkjet-printer-escpr

Interfaces:
> Printers can be managed using the CUPS web-interface on http://localhost:631/ or using the KDE interface, I will use the KDE interface for the configurations.

Configurations:
> System Settings > Printers > Add printer  
The settings applied for the discovered Printers on the network did NOT work for me.  
Instead, I configured manually using AppSocket/HP JetDirect, with the printer static local IP (192.168.1.98:9100).   
After this, if the EPSON drivers were installed, simply select the printer model (Epson Stylus SX430).  
Disable sharing, enable default printer and print test page to verify.  

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/CUPS  
https://wiki.archlinux.org/index.php/CUPS#KDE  
https://wiki.archlinux.org/index.php/CUPS_printer_sharing  
</sup></sub> 

##### SANE

To use SANE with an EPSON printer I'm going to use "Image Scan! for Linux".
> sudo packer -S iscan iscan-plugin-network

Next just add the printer IP to the config file:
> sudo nano /etc/sane.d/epkowa.conf  
net 192.168.1.98

<sub><sup>
References:
https://wiki.archlinux.org/index.php/sane#For_Epson_hardware
</sup></sub> 

##### Activate stand-by mode on HDDs

Normaly we would use the `hdparm` tool, but this tool does not seem to work well with my Westrem Digital hard drives. It is possible to activate standby on demand with the command:

> sudo hdparm -y /dev/sdc  

and it is also possible to check the disk state:

> sudo hdparm -C /dev/sdc  

But, it does not work well when we try to activate with X time of inactivity, for example:

> sudo hdparm –S 5 /dev/sdc  

For me the solution lies on using the tool `hd-idle` instead:

> packer –S hd-idle  
systemctl enable hd-idle  
systemctl start hd-idle  
systemctl status hd-idle  
hd-idle -i 0 -a sdb -i 600 -a sdc -i 600 –a sdd –I 1200  

```
Example:
hd-idle -i 0 -a sda -i 300 -a sdb -i 1200

This example sets the default idle time to 0 (meaning hd-idle will never try to spin down a disk), then sets explicit idle times for disks which have the string "sda" or "sdb" in their device name. 
```
<sub><sup>
References:  
http://hd-idle.sourceforge.net/  
http://askubuntu.com/questions/196473/setting-sata-hdd-spindown-time  
http://www.spencerstirling.com/computergeek/powersaving.html#harddrive  
</sup></sub>

##### Android MTP protocol

To use the Media Transfer Protocol which is used by many mobile phones and media players we need to give dolphin this ability. To do so, we simply have to install the following package:

> sudo pacman -S kio-mtp

<sub><sup>
References:
https://wiki.archlinux.org/index.php/MTP 
</sup></sub>

### Installs

##### Firefox and H.264 codec support:
> sudo pacman -S firefox firefox-i18n-pt-pt gstreamer gst-plugins-good gst-libav  
Test support: http://www.quirksmode.org/html5/tests/video.html

Remove some of the password manager functionality:

> Go to `/home/filipe/.mozilla/firefox/XXXXXXXX.default/`  
Create the folder `chrome`  
Create the file `userChrome.css` inside the `chrome` folder.  
Edit the file with the following options:  
```
#removeAllSignons {display: none;}   
#removeSignon {display: none;}   
#togglePasswords {display: none;}   
```

Set as default browser on xdg-open:

> xdg-mime default firefox.desktop x-scheme-handler/http  
xdg-mime default firefox.desktop x-scheme-handler/https  
xdg-mime default firefox.desktop text/html  
xdg-settings set default-web-browser firefox.desktop

##### Firewall:
> sudo pacman -S ufw  
sudo packer -S kcm-ufw  
sudo ufw enable  
sudo systemctl enable ufw.service  
sudo systemctl start ufw.service

##### Antivirus:
> sudo pacman -S clamav  
sudo packer -S clamtk  
sudo freshclam  
sudo systemctl enable clamd.service  
sudo systemctl start clamd.service  

##### XScreensaver:
> sudo pacman -S kdeartwork-kscreensaver xscreensaver  

##### Screenshots
> sudo pacman -S kdegraphics-ksnapshot 

##### PDF Reader
> sudo pacman -S kdegraphics-okular 

##### Notepad
> sudo pacman -S kdesdk-kate 

Configurations:
> Kate > Configurations > Activate console plugin.   
Kate > Configurations > Appearence > Borders > Activate all 

##### KDE Sudo:
> packer -S kdesudo

##### Calculator
> packer -S extcalc

##### Libreoffice:

I prefer `Libreoffice` over `calligra` due to compatiblity and similarity to MS office.

> sudo pacman -S libreoffice-fresh libreoffice-fresh-pt  
sudo pacman -S hunspell-en  
sudo packer -S hunspell-pt_pt  
sudo packer -S libreoffice-extension-languagetool  

##### 7Zip:
> sudo pacman -S p7zip zip unzip unrar kdeutils-ark

##### Imageviewer:

I prefer `photoqt` over `gwenview` due to its simplicity.

> sudo packer -S photoqt

##### QtCreator

> sudo pacman -S gdb valgrind qt5-doc qtcreator

##### Brackets

I prefer `brackets` over `sublimetext` because it is open source.  
`brackets` also has emmet support, an essencial plugin.  
Other alternative would be to use `vim` or `emacs`, but I like GUI.

> sudo packer -S brackets-bin

##### LAMP

After the instalation, just run, there is no need to enable this at boot:
> sudo systemctl start httpd mysqld

##### Apache

> sudo pacman -S apache  

By default, it will serve the directory `/srv/http`

##### PHP

> sudo pacman -S php php-apache

Configure PHP:

> sudo nano /etc/httpd/conf/httpd.conf  
> Comment this line: LoadModule mpm_event_module modules/mod_mpm_event.so  
> Add this line: LoadModule mpm_prefork_module modules/mod_mpm_prefork.so  
> Add this line: LoadModule php5_module modules/libphp5.so  
> Add this line: Include conf/extra/php5_module.conf  

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/Apache_HTTP_Server#PHP
</sup></sub>

##### Mysql

> sudo pacman -S mariadb 

After the instalation, initialize MySQL data directory and creates the system tables:

> sudo mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql  

Start MySQL:

> sudo systemctl start mysqld

Secure instalation:

> sudo mysql_secure_installation  

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/MySQL#Installation
</sup></sub>

##### Postgre

Install Postgre:

> sudo pacman -S postgresql  

Configurate the Postgres user:

> passwd postgres  
su - postgres -c "initdb --locale pt_PT.UTF-8 -D '/var/lib/postgres/data'"

Now, either start using the postgres user, or create a new one:

> createuser --interactive  
createdb myDatabaseName -U username

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/PostgreSQL
</sup></sub>

##### phpmyadmin

> sudo pacman -S phpmyadmin php-mcrypt

Configure phpmyadmin:

> sudo nano /etc/php/php.ini  
> Uncomment:  
extension=mysqli.so  
extension=mcrypt.so  
extension=zip.so  
> Add `/etc/webapps` to `open_basedir`  
> Result: open_basedir = /srv/http/:/home/:/tmp/:/usr/share/pear/:/usr/share/webapps/:/etc/webapps/

Create the following config file:

> sudo nano /etc/httpd/conf/extra/phpmyadmin.conf

```
Alias /phpmyadmin "/usr/share/webapps/phpMyAdmin"  
<Directory "/usr/share/webapps/phpMyAdmin">  
    DirectoryIndex index.php  
    AllowOverride All  
    Options FollowSymlinks  
    Require local  
</Directory>  
```

Also, if you wish to restrict phpmyadmin to local access you can use the following config. But remember, if you want to be really safe, I would advise to remove it completely. Also, with this config, you will only be able to access by typing  `127.0.0.1` not `localhost`.

```
Alias /phpmyadmin "/usr/share/webapps/phpMyAdmin"
<Directory "/usr/share/webapps/phpMyAdmin">
    DirectoryIndex index.php
    AllowOverride All
    Options FollowSymlinks
    Require all granted
    Order Deny,Allow
    Deny from all
    Allow from 127.0.0.1
</Directory> 
```

And finnaly include it:

> sudo nano /etc/httpd/conf/httpd.conf

> \#phpMyAdmin configuration  
Include conf/extra/phpmyadmin.conf

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/PhpMyAdmin#Configuration
</sup></sub>

Install mysql-workbench:

> sudo pacman -S mysql-workbench gnome-keyring

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/PhpMyAdmin#Configuration
</sup></sub>

##### Owncloud

With the above instalation of LAMP already working:

> sudo pacman -S owncloud php-intl php-mcrypt php-xcache exiv2 

Owncloud configuration:

In `/etc/php/php.ini` enable:
```
gd.so
iconv.so
xmlrpc.so
zip.so
bz2.so
curl.so
intl.so
mcrypt.so
openssl.so
pdo_mysql.so
mysql.so
exif.so
```

Copy the Apache configuration file to its configuration directory:

> cp /etc/webapps/owncloud/apache.example.conf /etc/httpd/conf/extra/owncloud.conf

And include it at the bottom of `/etc/httpd/conf/httpd.conf`:

> Include conf/extra/owncloud.conf

Now the easy way to set the correct permissions is to copy and run this script. Replace the `ocpath` variable with the path to your ownCloud directory, and replace the `htuser` variable with your own HTTP user. If the `data` directory does not yet exist, please create it first. (Script extracted from owncloud wiki)

```
#!/bin/bash
ocpath='/usr/share/webapps/owncloud'
htuser='http'

find ${ocpath}/ -type f -print0 | xargs -0 chmod 0640
find ${ocpath}/ -type d -print0 | xargs -0 chmod 0750

chown -R root:${htuser} ${ocpath}/
chown -R ${htuser}:${htuser} ${ocpath}/apps/
chown -R ${htuser}:${htuser} ${ocpath}/config/
chown -R ${htuser}:${htuser} ${ocpath}/data/

chown root:${htuser} ${ocpath}/.htaccess
chown root:${htuser} ${ocpath}/data/.htaccess

chmod 0644 ${ocpath}/.htaccess
chmod 0644 ${ocpath}/data/.htaccess
```

Now we just need to activate xcache, for that remove the comment on xcache.ini

> sudo nano /etc/php/conf.d/xcache.ini

And add the entry on config.php:

> sudo nano /etc/webapps/owncloud/config/config.php
'memcache.local' => '\\OC\\Memcache\\XCache'

Finnaly we have to give owncloud access to `/dev/urandom`.
To do so, just attach :/dev/urandom (no slash at the end) to open_basedir in php.ini.
For easy access just open the file on kate, and find the correct line.

> kdesu kate /etc/php/php.ini

Everything should be working. Now we are going to setup HTTPS access:

> sudo pacman -S openssl

Generate server key and certificate:

> cd /etc/httpd/conf  
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out server.key  
chmod 600 server.key  
openssl req -new -sha256 -key server.key -out server.csr  
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt  

Then, in `/etc/httpd/conf/httpd.conf`, uncomment the following three lines:

> LoadModule ssl_module modules/mod_ssl.so  
LoadModule socache_shmcb_module modules/mod_socache_shmcb.so  
Include conf/extra/httpd-ssl.conf  
Include conf/extra/httpd-vhosts.conf

Now lets setup a virtualhost, but first, we need to comment the existing one in (this can also be done with the purpose of stopping owncloud from taking control of the port 80):

> sudo nano /etc/httpd/conf/extra/owncloud.conf

```
#<VirtualHost *:80>
#    ...
#</VirtualHost>
```

Now we add a new virtualhost that uses HTTPS:

> sudo nano /etc/httpd/conf/extra/httpd-vhosts.conf

```
<VirtualHost *:443>
    ServerName OwnCloud
    DocumentRoot /usr/share/webapps/owncloud

    SSLEngine On
    SSLCertificateFile /etc/httpd/conf/server.crt
    SSLCertificateKeyFile /etc/httpd/conf/server.key

    ErrorLog /var/log/httpd/owncloud.info-error_log
    CustomLog /var/log/httpd/owncloud.info-access_log common
</VirtualHost>
```

If you want to map to another port, for example 1337, just change the above virtualHost port and `nano /etc/httpd/conf/httpd.conf` and add `Listen 1337`.

Now, this should be working for HTTPS only, for an easier access, we can redirect the port 80 to the port 443 with the following host in `/etc/httpd/conf/extra/httpd-vhosts.conf`:

```
<VirtualHost *:80>
   ServerName OwnCloud
   Redirect permanent / https://LOCAL.OR.PUBLIC.IP:80/
</VirtualHost>
```

Finnaly, dont forget to forward these TCP ports on your router and local firewall to enable public access.

XCache Warning:  
"XCache opcode cache will not be cleared because "xcache.admin.enable_auth" is enabled."

To solve this just go to `/etc/php/conf.d/xcache.ini` and add the line:  
> xcache.admin.enable_auth=off

The log is located in:  
/usr/share/webapps/owncloud/data/owncloud.log

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/OwnCloud  
https://wiki.archlinux.org/index.php/Apache_HTTP_Server#TLS.2FSSL  
http://serverfault.com/questions/528210/bind-apache-ssl-port-with-different-port-with-same-openssl-port-443  
https://doc.owncloud.org/server/8.0/admin_manual/installation/installation_wizard.html#setting-strong-directory-permissions  
</sup></sub>

##### Dropbox

> sudo packer -S dropbox

On a multiboot configuration it would be best to choose a partition that is common to all installed OS.

Specific folders on different disks with symlink:  
> sudo ln -s /media/Dados1/Músicas/ /home/filipe/Dropbox/Privado/Músicas  
Warning: the forlder “Musicas” on the destination (dropbox) should NOT be already created.  

##### Syncthing

Syncthing replaces proprietary sync and cloud services with something open, trustworthy and decentralized.  
> sudo pacman -S syncthing

Inotify (inode notify) is a Linux kernel subsystem that acts to extend filesystems to notice changes to the filesystem, and report those changes to applications. 
> packer -S  syncthing-inotify

The syncthing-inotify service requires syncthing so you don't have to start/enable syncthing seperately. 
> systemctl --user enable syncthing-inotify.service  
systemctl --user start syncthing-inotify.service

Open local ports on the firewall:
> Port 22000/TCP - Sync Protocol Listen Address  
Port 21025/UDP - Discovery broadcasts on IPv4

Make sure your user has permissions to sync the selected folders.

I also tried the package `syncthing-gtk`, but the tray function does NOT work properly.  
There are some tray programs being built in Qt, check later:
https://forum.syncthing.net/t/osx-syncthingtray-update-qt-based-traybar/5391

They have a very good documentation:
http://docs.syncthing.net/index.html

##### Truecrypt

> sudo pacman -S truecrypt

Auto-Mount the truecrypt container on boot using KDE:  
> System configurations > Start and stop > Add program > System menu and select TrueCrypt.  
mkdir /media/Trabalho  
nano /home/filipe/.config/autostart/truecrypt.desktop  
```
Exec=truecrypt -t /PATH-TO-VOLUME /media/Trabalho -p 'PASSWORD' -k '' --protect-hidden=no --mount-options=readonly --fs-options=iocharset=utf8  
Terminal=true
```

Avoid entering the root password at boot:
> EDITOR=nano visudo  
Add at the end:  
filipe ALL=NOPASSWD: /usr/bin/truecrypt  

Auto-Unmount the truecrypt container using KDE:

Create the following stript:
> sudo nano ~/.kde4/shutdown/truecryptshutdown

```
#!/bin/bash
/usr/bin/truecrypt -d
```

Make it executable:
> sudo chmod 755 ~/.kde4/shutdown/truecryptshutdown

Notice that the script was added in:
> System configurations > Start and stop

<sub><sup>
References:  
http://okomestudio.net/biboroku/?p=303  
http://andryou.com/truecrypt/docs/command-line-usage.php  
http://ubuntuforums.org/showthread.php?t=1646881  
http://medienvilla.com/index.php?id=236#linux_script  
http://ubuntuforums.org/showthread.php?t=1924213  
http://askubuntu.com/questions/68327/why-does-truecrypt-ask-for-administrator-password  
http://askubuntu.com/questions/88523/creating-a-mount-point-if-it-does-not-exist  
</sup></sub>

##### XMPP

> sudo pacman -S pidgin pidgin-otr

Configurations:  
> Activate the following plugins:  
Voice and video configuration  
Offline messages emulation  
History  
Off the record messaging  

Tested:  
> pidgin-extprefs - Useless  
purple-plugin-pack - Useless  
pidgin-encryption - OTR is better for IM  

Note:  
KDE will restore pidgin state on boot.

##### Skype

> sudo pacman -S skype  
sudo packer -S skype-secure

If the sound is not working try:
> As the main user, copy /etc/pulse/default.pa to ~/.pulse/default.pa and add:  
load-module module-native-protocol-tcp auth-ip-acl=127.0.0.1  
As the skype user, create ~/.pulse/client.conf and add:  
default-server = 127.0.0.1   
reboot  

Distorted sound:
> If you get distorted sound in skype, try adding PULSE_LATENCY_MSEC=60 to your env before starting skype.    
Something like 'export PULSE_LATENCY_MSEC=60' in .bashrc, for example.  

Skype stops video playback with notifications:  
> nano /etc/pulse/default.pa  
Comment the following line:  
load-module module-cork-music-on-phone  

Test skype access:
> su -  
nano /etc/passwd  
replace /sbin/nologin with /bin/bash  
passwd _skype  
su _skype  
Do whatever access tests needed, for example:  
ls /main_user_folder  
ls /internal_discs (deny access already set on the mount masks in fstab)  
Lock the _skype account again:  
su -  
passwd -d _skype  
nano /etc/passwd  
replace /bin/bash with /sbin/nologin  
Check groups with: groups _skype (should be video, audio, _skype)  

Configurations:  
> Add skype to startup in KDE  
Options > General > Start minimized  
Options > General > Style GTK+  
Options > Message > Animated Icons  
Options > Video > Disable webcam auto-brightness  

Links and downloads:  
There are ways to make the links open on the same browser, but I will not enable this.  
Same goes for the downloads, they will be transfered to the _skype account folder, I will move them manually.

Other types of protection:  
Using `apparmor` and `tomoyo` requires kernel recompilation.  
Using `docker` and `sandfox` seem outdated or lack instructions.  

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/skype#Use_Skype_with_special_user  
http://lightrush.ndoytchev.com/random-1/debiansqueezefixespreventskypefrompausingaudiovideoplayback  
https://forums.gentoo.org/viewtopic-t-997098.html?sid=531888f644a8f99aac8d4bc0260ddf83  
</sup></sub>

##### TOR

We can use TOR with an existing browser (not recommended) simply by installing and running:

> sudo pacman -S tor  
tor

Then we open the browser and set the SOCKS proxy to `localhost` with port `9050`  
For ease we can install a plugin to toggle proxys.

A better alternative is to use to TOR Browser. I did not have any luck in installing `tor-browser-en`, it keept falling on verifying the keys even after I added them to the keyring (other users complain of the same problem on the AUR). An alternative to this, is to simply download the bundle directly from https://www.torproject.org/download/ extract and double click the provided shortcut on the folder to run the browser.

##### Steam

> sudo pacman -S steam

Close to tray:
> sudo nano /etc/environment
Add:
STEAM_RUNTIME=1
STEAM_FRAME_FORCE_CLOSE=1
Alternatively, start steam with:
STEAM_FRAME_FORCE_CLOSE=1 steam

The first line tells steam to use its libraries instead of the system. They might be more outdated, but we guarantee that there will be no problems. The second line enables close to tray.

Start silently:
> Enable the startup at boot on the steam client and then add the `-silent` to entry using KDE.

OpenGL problem:
> rm /home/filipe/.local/share/Steam/ubuntu12_32/steam-runtime/i386/lib/i386-linux-gnu/libgcc_s.so.1
rm /home/filipe/.local/share/Steam/ubuntu12_32/steam-runtime/i386/usr/lib/i386-linux-gnu/libstdc++.so.6
Alternatively, start steam with:
DRI_PRIME=1 LD_PRELOAD="/usr/lib/libstdc++.so.6 /usr/lib32/libstdc++.so.6 /usr/lib/libgcc_s.so.1 /usr/lib32/libgcc_s.so.1" ~/.local/share/Steam/ubuntu12_32/steam-runtime/run.sh steam

Instalation notes:  
When installing steam will give the following advices:

If you are having problems with the steam license, remove .steam and .local/share/Steam  
If you are running x86_64, you need the lib32 opt depends for your driver.  

> lib32-mesa-dri: for open source driver users  
lib32-catalyst-utils: for AMD Catalyst users  
lib32-nvidia-utils: for NVIDIA proprietary blob users  
lib32-alsa-plugins: for pulseaudio on some games  

##### Clementine

> sudo pacman –S clementine

Configurations:  
> Add library.  
Make the image on the left bigger. (Start a music and right-click)  
Preferences > Reproduction > Disable animations, Fade-in, Fade-out  
Preferences > Notifications > Personalized, 3 seconds, bottom right corner.   

##### Diagrams

There are programs like DIA or Violet for UML diagrams.  
But I prefer to use the https://www.draw.io/ website.  
To me, has better usability, there is no need for account and supports offline mode.

##### VirtualBox

> sudo pacman -S virtualbox virtualbox-host-modules virtualbox-guest-iso

To enable the necessary kernel modules at boot we need to create a static config file under `/etc/modules-load.d/`, these files will be automatically loaded by [udev](https://wiki.archlinux.org/index.php/Udev)

> sudo nano /etc/modules-load.d/virtualbox.conf  
vboxdrv  
vboxnetadp  
vboxnetflt  
vboxpci  

The only obrigatory module is `vboxdrv` but we need the other modules to use aditional functionalities like bridged networking and PCI passthrough.

At every kernel update we will have to reload the modules manually by doing:

> sudo modprobe vboxdrv

Alternatively, install DKMS along with the DKMS version of virtualbox.

To use shared folders, first, we create a folder on the host OS and select the folder on the VM settings.
Once we start our Linux VM we need to mount the folder. To do so, we can use the following command:

> sudo mount -t vboxsf -o rw,uid=1000,gid=1000 share '/home/filipe/Shared'

Find the user uid with `cat /etc/passwd` and the gui with `cat /etc/groups` or `group username`  

To enable at boot, simply copy the above command to the file `/etc/rc.local`. Remember to change the uid and gid if needed, this is to give your user permission to read and write on this folder.

Finally, if you use the aforementioned "Host-only" or "bridge networking" feature, make sure net-tools is installed. VirtualBox actually uses `ifconfig` and `route `to assign the IP and route to the host interface configured with VBoxManage hostonlyif or via the GUI in Settings > Network > Host-only Networks > Edit host-only network (space) > Adapter. 

> sudo pacman -S net-tools

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/VirtualBox  
https://forums.virtualbox.org/viewtopic.php?t=15868
</sup></sub>

##### Lancelot

> sudo pacman -S kdeplasma-addons-applets-lancelot  
Activate by adding the new icon from the elements.  

<sub><sup>
References:  
http://www.archlinuxuser.com/2014/04/change-default-kde-start-menu-with.html  
https://www.kubuntuforums.net/showthread.php?59851-KDE-Application-Launchers  
</sup></sub>

##### Themes

Caledonia: https://aur.archlinux.org/packages/caledonia-bundle/  
Logon: http://kde-look.org/content/show.php/ArchPrecise-KDM-Theme?content=161886  
Splash: http://kde-look.org/content/show.php/modern+Arch+Linux?content=164279

Other:  
http://kde-look.org/content/show.php/Modern+KDM+Archlinux?content=163475  
http://kde-look.org/content/show.php/Modern+KDE+4+Splash+Screen?content=163449

##### Video Editor

> sudo pacman –S kdenlive

I've tested `Avidemux`, `Cinelerra` and `Open Shot` but I prefer `kdenlive`.

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/List_of_applications/Multimedia#Graphical_3
</sup></sub>

##### File Compare

> sudo pacman -S meld

I tested `Kompare` but it does not work as well.

##### Subtitle Editor

> sudo pacman -S gaupol

##### SSH/FTP/VNC/NX Client

> sudo pacman -S remmina libvncserver

##### Torrent Client

> sudo packer -S qbittorrent popcorntime-bin

KDE will restore qtorrent state on boot.  

##### Caffeine 

> sudo packer -S caffeine-ng

Used to prevent the screensaver to kickin due to inactivity, like watching an online video.

##### Image Editor 

> sudo packer -S gimp imagemagick

Imagemagick is a command line tool, some examples here:  
http://askubuntu.com/questions/246647/jpeg-files-to-pdf  
http://forum.linuxcareer.com/threads/1644-Batch-image-resize-using-linux-command-line

Activate single window mode on gimp.

##### Partition manager

> sudo pacman -S partitionmanager  

##### Surveillance

I was not able to find any decent surveillance software. I tested:  
Zoneminder - It is just too massive for a single camera and forces the use of apache.  
Xeoma - I was not able to run this.  
To test: gSpy, bluecherrydvr, Ubiquity's Unifi Video

In Windows I used `iSpy` and it does a very good job.  

##### Equivalent of `tracert` in Windows, `traceroute`

> sudo pacman -S traceroute

##### Multiboot USB

This is for Linux distros only, for Windows use YUMI on Windows.

> packer -S multibootusb

##### Wireshark

> sudo pacman -S wireshark-qt

Running Wireshark as root is insecure.

`/usr/bin/dumpcap` is the only process that has privileges to capture packets.  
`/usr/bin/dumpcap` can only be run by root and members of the `wireshark` group. 

Add yourself to the `wireshark` group:

> gpasswd -a filipe wireshark

To make your session aware of this new group without having to log in:

> newgrp wireshark && groups

##### Crop PDFs

> packer -S briss

##### OCR Tool

I've tested: gscan2pdf, gimagereader, ocropy...  
I have to say that tesseract is an absolute shit.  

Use instead:  
> sudo pacman -S ocrfeeder cuneiform

### KDE Look & feel

Move "Tool Box" on the desktop to the right top corner.  
Configurate tray > Always show > All except the two notification icons.  
Configurate taskbar > No groups, manually ordered.  
Configurate system configurations > Classic tree, remove detailed tips, expand first level.  
Configurate desktop > Disposition: Folder.  
Configurate startmenu > change the icon to [one of these](http://gabriela2400.deviantart.com/art/Arch-Linux-Start-Icons-175557586)  
Dolphin > Adjust window properties:  Show in details, order by name, show folders first, show preview, show hidden files. Additional Information: Name, Size, Date, Type, Location, permissions, owner. Apply to all folders, use by default.  
Dolphin > Configure Dolphin: Change default start up, show location bar, show filter, doble-click to open, delete files from garbage after 7 days show space available information.   
Dolphin > Right click on a drive > Icon size > Large.  
Dolphin > Right click on top menu > Icon size > Large.  
Dolphin > Icon size 32px (drag lower bar)  
System configurations > Add account image.  
System configurations > Screen borders > Lower right, Show Screen  
System configurations > Gobal hotkeys > KDE Sessions > Lock Screeb with Windows+L  
System configurations > Display and Screen > Protector > 5 min to init, 300 sec to ask pass, select screensaver  
VLC > Enable multiple instances
VLC > Make the player bar full with on full-screen

### System cleanup

##### Check logs

> systemctl --failed  
> nano /var/log/pacman.log  
> journalctl -p 0..3 -xn  

##### Find and remove orphan packages

> sudo pacman -Rns $(pacman -Qtdq)

If there is a packages that should not be in here, it is probably because it was installed as a depenency when it should have been instaled explicitly.

Check if that is the case with:

> pacman -Qi \<pkg>

And mark it as explict with:

> pacman -D --asexplicit \<pkg>

##### Improving database access speeds

> pacman-optimize

##### Clean the package cache (/var/cache/)

Pacman stores its downloaded packages in `/var/cache/pacman/pkg/` and does not remove the old or uninstalled versions automatically, therefore it is necessary to deliberately clean up that folder periodically to prevent such folder to grow indefinitely in size. 

> paccache -ruk0

##### Clean systemd journal (/var/log/)

> sudo journalctl --vacuum-time=10d 

Cleans everything except ne previous 10 days.

##### Manually clean the home directory

In the $HOME folder there are files created by many different programs, and as such there is no easy way to manage them all. The best way is to search manually and double check the name of folders and files with packages name with `pacman -Qs <pkg>`

~/ - Probably some useless empty folders  
~/.config/ - Where apps stores their configuration  
~/.cache/ - Cache of some programs may grow in size  
~/.local/share/ - Old files may be lying there  

Also, there is an application that tries to acomplish this, called mundus, but it did not run on my machine.

##### Find lost files

A simple bash script that shows users "lost" files on their Arch Linux systems. "Lost" in this context means those files that are not owned by an installed Arch Linux package.

> sudo packer -S lostfiles  
> sudo lostfiles

Inspect and manually delete the files.

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/System_maintenance  
https://wiki.archlinux.org/index.php/Pacman_tips#Removing_orphaned_packages  
https://wiki.archlinux.org/index.php/Pacman#Cleaning_the_package_cache  
https://wiki.archlinux.org/index.php/Improve_pacman_performance#Improving_database_access_speeds  
https://aur.archlinux.org/packages/lostfiles/  
</sup></sub>

### Random problems and notes

##### Misbehaved packages

> nano /etc/pacman.conf  
IgnorePkg = xyz

##### Not-found services

> sudo systemctl -t services -a  
Not really a problem: https://ask.fedoraproject.org/en/question/49946/how-to-delete-a-systemd-service/

##### Black Screen

KDM might have crashed, due to some recent changes, check why in `/var/log`

> CRTL+ALT+F1  
sudo systemctl stop kdm.service  
sudo systemctl start kdm.service

##### Duplicated Open-with

Remove duplicated items on the open-with context menu: 

> cd ~/.local/share/applications  
rm ...

##### Modify shortcuts

> Either search for .desktop files or go to `/user/share...`  
Edit the `Exec:` line.

https://wiki.archlinux.org/index.php/Desktop_entries

##### Install gstreamer0.10-good-plugins

```
(gconftool-2:2335): GConf-WARNING **: Client failed to connect to the D-BUS daemon:
/usr/bin/dbus-launch terminated abnormally with the following error: No protocol specified
Autolaunch error: X11 initialization failed.
```
https://bbs.archlinux.org/viewtopic.php?pid=1188169#p1188169

##### Dolphin console

Keep it hidden, otherwise it will litter the console history with `cd` commands.

##### Watchdog

At shutdown the system prints: kernel: watchdog watchdog0: watchdog did not stop!  
This is normal: https://bbs.archlinux.org/viewtopic.php?pid=1195597#p1195597

##### Microphone Playback

> alsamixer  
Select sound card with F6  
Select reproduction devices, if not already  
Make the value of Mic 0  

##### Microphone Echo

This is diferent from the above problem, this happens when we receive audio, or we speak. It repeats alot of times.

> alsamixer  
Select sound card with F6  
Select record devices  
Make the value of PCM 0  

##### Clementine does not close

https://github.com/clementine-player/Clementine/issues/1728

##### Remmina closes VNC connection without notice

This might be because the color preferences are not the same on the server and on the client.  
Try changing the color preferences of the VNC connection, to match the VNC server.

##### 0B Removable Media in device notifier

Normally this is a floppy drive, you can confirm with the presence or absence of `/dev/fd0`

But, if I dont have a floppy drive, why is this here? Simple, the floopy drive hardware is not actually capable of being auto detected; so, this is something that has to be configured in the system BIOS. You have to manually tell the BIOS what type of floppy you have, and it in turn tells the OS.

So you need to go into your BIOS and tell it that you have no floppy.

UNTESTED: Another way around this is to blacklist the floppy drive in the kernel modules.

##### KDE restores volume to 100%

> https://bugs.kde.org/show_bug.cgi?id=324975

##### GRUB Message

If GRUB flashes a message for a split second at boot, it is possible that the message is like the following:  
> Welcome to Grub!  
error: file '/boot/grub/locale/pt.mo' not found

To fix this we simply need to copy one of the existing configs to the missing one:  

> sudo cp /boot/grub/locale/pt_BR.mo /boot/grub/locale/pt.mo

##### Kernel Missing firmwares

Search for it and simply install:

> packer -S aic94xx-firmware wd719x-firmware

##### MySQL Workbench Warning

> MySQL WorkBench is an Oracle product, and will therefore warn about the version numbers used by MariaDB. The error message starts with a text similar to this: "Incompatible/nonstandard server version or connection protocol detected (10.0.13)" Select "Continue Anyway" to continue connecting.

##### Libreoffice does not autocorrect

Check if you have a blue tick on the combobox in:
> Tools -> Options -> Language Settings -> Languages  

http://askubuntu.com/questions/203727/libreoffice-spell-checker-doesnt-work

##### Backlight

Do NOT install the package `asus-kbd-backlight` because it is no longer required, current KDE desktop can control the keyboard brightness with the native facilities. 

##### System Hierarchy

https://wiki.archlinux.org/index.php/Arch_filesystem_hierarchy
https://techbase.kde.org/KDE_System_Administration/KDE_Filesystem_Hierarchy

##### Create Systemd Service

This script does NOT work with the above auto-mount using KDE.
If you want to make this work, make sure you also mount with systemd.

> sudo nano /usr/lib/systemd/system/truecryptshutdown.service

```
[Unit]
Description=Truecrypt Unmount
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/true
ExecStop=/usr/bin/truecryptshutdown

[Install]
WantedBy=multi-user.target
```

Now create your script:

> sudo nano /usr/bin/truecryptshutdown

```
#!/bin/bash
/usr/bin/truecrypt -d
```

Change permissions:
> sudo chmod 755 /usr/bin/truecryptshutdown

To test the script:
> sudo systemctl status truecryptshutdown
sudo systemctl start truecryptshutdown

Since this script is only aimed to work uppon exit, use restart to test:
> sudo systemctl restart truecryptshutdown

<sub><sup>
References: 
http://unix.stackexchange.com/questions/39226/how-to-run-a-script-with-systemd-right-before-shutdown
</sup></sub>

##### Give pacman some color

> Uncomment the "Color" line in /etc/pacman.conf

##### Owner, group, permissions summary

###### Groups:

Create group:
> groupadd groupname

Delete group:
> groupdel groupname

Change group GID:
> groupmod -g NEW_GID groupname

List primary and secondary groups from user:
> id  
groups  
cat /etc/group | grep filipe  
getent group <groupname>

###### Users:

Create User:
> useradd username 

Delete User:
> userdel username

Append existing user to group:
> usermod -a -G username group

Remove user from group:
(The new group config will be assigned at the next login)
> gpasswd -d user group

Assign primary group to user:
> usermod -g primarygroupname username

Assign primary group to user:
> usermod -G secondarygroupname username

###### Permissions:

Change set owner and group:
> chown user:group [file/dir]

`chown` does the same that `chgrp`: 
http://unix.stackexchange.com/questions/164853/what-is-the-point-of-chgrp

Each file and directory has three user based permission groups:
 ```
owner - The Owner permissions apply only the owner of the file or directory, they will not impact the actions of other users.
group - The Group permissions apply only to the group that has been assigned to the file or directory, they will not effect the actions of other users.
all users - The All Users permissions apply to all other users on the system, this is the permission group that you want to watch the most.
 ```
 
Each file or directory has three basic permission types:
 ```
read - The Read permission refers to a user's capability to read the contents of the file.
write - The Write permissions refer to a user's capability to write or modify a file or directory.
execute - The Execute permission affects a user's capability to execute a file or view the contents of a directory.
 ```

The special permissions flag can be marked with any of the following:
```
_ - no special permissions
d - directory
l - The file or directory is a symbolic link
s - This indicated the setuid/setgid permissions. This is not set displayed in the special permission part of the permissions display, but is represented as a s in the read portion of the owner or group permissions.
t - This indicates the sticky bit permissions. This is not set displayed in the special permission part of the permissions display, but is represented as a t in the executable portion of the all users permissions
 ```

When applying permissions to directories on Linux, the permission bits have different meanings than on regular files:
 ```
The write bit allows the affected user to create, rename, or delete files within the directory, and modify the directory's attributes
The read bit allows the affected user to list the files within the directory
The execute bit allows the affected user to enter the directory, and access files and directories inside
The sticky bit states that files and directories within that directory may only be deleted or renamed by their owner (or root)
```

##### IPV6 router solicitation

Check if the network manager is constantly making router solicitation.
> journalctl -p 0..3 -xn

Ther errors are like this:
```
Mar 13 11:50:42 NetworkManager[1106]: <error> [1426243842.961433] [rdisc/nm-lndp-rdisc.c:241] send_rs(): (eth0): cannot send router solicitation: -1.
Mar 13 11:50:43 NetworkManager[1106]: <error> [1426243843.959533] [rdisc/nm-lndp-rdisc.c:241] send_rs(): (wlan0): cannot send router solicitation: -1.
Mar 13 11:50:46 NetworkManager[1106]: <error> [1426243846.960535] [rdisc/nm-lndp-rdisc.c:241] send_rs(): (eth0): cannot send router solicitation: -1.
Mar 13 11:50:47 NetworkManager[1106]: <error> [1426243847.959683] [rdisc/nm-lndp-rdisc.c:241] send_rs(): (wlan0): cannot send router solicitation: -1.
Mar 13 11:50:50 NetworkManager[1106]: <error> [1426243850.962048] [rdisc/nm-lndp-rdisc.c:241] send_rs(): (eth0): cannot send router solicitation: -1.
Mar 13 11:50:51 NetworkManager[1106]: <error> [1426243851.959510] [rdisc/nm-lndp-rdisc.c:241] send_rs(): (wlan0): cannot send router solicitation: -1.
Mar 13 11:50:54 NetworkManager[1106]: <error> [1426243854.961697] [rdisc/nm-lndp-rdisc.c:241] send_rs(): (eth0): cannot send router solicitation: -1.
```

You have to disable IPv6 on the connection.  
To do so, go to:
> NetworkManager Tray > Edit Connections > Wired > Network name > Edit > IPv6 Settings > Method > Ignore/Disabled

I also tried the following method, and DOES NOT WORK:
http://askubuntu.com/questions/440649/how-to-disable-ipv6-in-ubuntu-14-04

##### Disable IPv6 on UFW

If when you click `IPv6 Support` in `System Settings > Network and Conectivity > Firewall` check box, and it changes the firewall status, try the "Restore Defauls" button, an then reset:

> Firewall Status - Yes  
IPv6 Support - No  
Outgoing Policy - Allow  
Incoming Policy - Deny  
Logging Level - Medium  
