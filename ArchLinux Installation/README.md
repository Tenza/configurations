# ArchLinux Instalation Notes

These notes were created for my desktop and notebook installations.  
The desktop uses a single SSD and the notebook (Asus UX51VZ) uses two SSD's on a RAID 0 configuration.  
Both have dualboot with Windows 7, and since both use SSD drives we will make some recommend optimizations.

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
Add to the SSD disk parameters: `discard`  
Update the SSD disk parameters: `realatime` for `noatime`  
Make sure the SWAP is not on `/mnt` and rather on `/`

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

##### Dualboot

First we install the package responsible for detecting other OS instalations:

> pacman -S os-prober

<sub><sup>
References:
https://wiki.archlinux.org/index.php/GRUB#Dual-booting
</sup></sub>

##### Windows already instaled:

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

> mount /mnt  
arch-chroom /mnt  
grub-mkconfig -o /boot/grub/grub.cfg  
grub-install /dev/**sda**

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

> useradd -m -G wheel -s /bin/bash filipe  
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

> sudo pacman -S mesa-dri xf86-video-intel nvidia bumblebee 

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

##### Install KDE

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

##### Mount on boot

> sudo pacman -S ntfs-3g  
sudo nano /etc/fstab

Desktop:

Besides mounting on boot, we will set access permissions to only the created user.  
I do this because I have some untrusted programs running on locked accounts, i.e Skype.

> /dev/sdb1               /media/Dados1   ntfs            defaults,uid=1000,dmask=027,fmask=137        0 0  
/dev/sdc1               /media/Dados2   ntfs            defaults,uid=1000,dmask=027,fmask=137        0 0  
/dev/sdd                /media/Dados3   ntfs            defaults,uid=1000,dmask=027,fmask=137        0 0  

Notebook:

> /dev/mp125p3               /media/Dados   ntfs            defaults,uid=1000,dmask=027,fmask=137        0 0

<sub><sup>
References:  
http://en.wikipedia.org/wiki/Fmask#Example  
https://wiki.archlinux.org/index.php/NTFS-3G  
http://www.omaroid.com/fstab-permission-masks-explained/  
http://askubuntu.com/questions/429848/dmask-and-fmask-mount-options
http://askubuntu.com/questions/113733/how-do-i-correctly-mount-a-ntfs-partition-in-etc-fstab
</sup></sub>

##### Uniform Look

To make GTK based applications look like Qt based, I prefer `oxygen-gtk` instead of `qtcurve`.

> sudo pacman -S oxygen-gtk2 oxygen-gtk3 kde-gtk-config

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Uniform_Look_for_Qt_and_GTK_Applications  
http://askubuntu.com/questions/50928/qtcurve-vs-oxygen-gtk-theme
</sup></sub>

##### Laptop tools

> sudo pacman -S acpid wireless_tools ethtool bluez-utils  
sudo packer -S laptop-mode-tools  
sudo systemctl enable laptop-mode  

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Laptop_Mode_Tools  
https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz#Powersave_management  
</sup></sub>

### Installs

##### Firefox and H.264 codec support:
> sudo pacman -S firefox firefox-i18n-pt-pt gstreamer gst-plugins-good gst-libav  
Test support: http://www.quirksmode.org/html5/tests/video.html

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

##### KDE tools:
> sudo pacman -S kdesdk-kate kdegraphics-okular kdeutils-kcalc

Configurations:
> Kate > Configurations > Activate console plugin.   
Kate > Configurations > Appearence > Borders > Activate all 

##### Libreoffice:

I prefer `Libreoffice` over `calligra` due to compatiblity and similarity to MS office.

> sudo pacman -S libreoffice-fresh libreoffice-fresh-pt  
sudo packer -S hunspell-pt_pt  

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

##### Dropbox

> sudo packer -S dropbox

On a multiboot configuration it would be best to choose a partition that is common to all installed OS.

Specific folders on different disks with symlink:  
> sudo ln -s /media/Dados1/Músicas/ /home/filipe/Dropbox/Privado/Músicas  
Warning: the forlder “Musicas” on the destination (dropbox) should NOT be already created.  

##### Truecrypt

> sudo pacman -S truecrypt

Mount the truecrypt container on boot:  
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

<sub><sup>
References:  
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

##### Steam

> sudo pacman -S steam

##### Others

> sudo packer -S qbittorrent partitionmanager-git

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

### Random problems

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
