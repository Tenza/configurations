#### Remove KDE4

KDE4 has been removed from the repos and is no longer suported. KDE5 is now in a stable state so this is prefered.
The past problems with the tray bar have been fixed and the applications are working as they should.

If you have KDE4 is installed, and since KDE4 and KDE5 cannot run together we need to remove it.  
Start by unloading the graphic shell, disable KDM from staring and remove KDE4 specific packages.

<pre>
systemctl isolate multi-user.target
systemctl disable kdm
pacman -Rnc kdebase-workspace 
pacman -Rns oxygen-gtk2 oxygen-gtk3-git kde-gtk-config-kde4
</pre>

#### KDE Tray system

Since KDE5 uses a new tray system, we need to change `QSystemTrayIcon` to `StatusNotifierItems` the package that does this is `sni-qt` and we need to install `libindicator` packages as well.

<pre>
packer -S gtk-sharp-2 libdbusmenu-gtk2 libdbusmenu-gtk3 libindicator-gtk2 libindicator-gtk3
packer -S libappindicator-gtk2
packer -S libappindicator-gtk3
packer -S sni-qt lib32-sni-qt
pacman -S kde-gtk-config
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/KDE
https://wiki.archlinux.org/index.php/Plasma  
https://wiki.archlinux.org/index.php/SDDM  
http://www.linuxveda.com/2015/02/27/how-to-install-kdes-plasma-5-on-arch-linux/
</sup></sub>

#### PulseAudio Audiophile

By default, PulseAudio (PA) uses very conservative settings. This will work fine for most audio media as you will most likely have 44,100Hz sample rate files. However, if you have higher sample rate recordings it is recommended that you increase the sample rate that PA uses.

<pre>
nano /etc/pulse/daemon.conf
add: default-sample-format = s32le 
add: default-sample-rate = 96000 
add: resample-method = speex-float-5 
</pre>

For the most geniune resampling at the cost of high CPU usage (even on 2011 CPUs) you can add: 

<pre>
nano /etc/pulse/daemon.conf
resample-method = src-sinc-best-quality 
</pre>

If you are having problems with the channels set by pulseaudio, you can set them manually by adding:

<pre>
nano /etc/pulse/daemon.conf
default-sample-channels = 3
default-channel-map = front-left,front-right,lfe
</pre>

<sub><sup>
References: 
http://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Audiophile/
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

After we reset these permissions, we might have problems on the Windows side.  
If we get "Access Denied" to the disk, just change the permissions of the "Everyone" user.

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

##### Hibernate

To have the hibernate functionality we have to add some kernel parameters as well as some hooks to the initramfs.  
But first, because I'm using a file swap intead of a partition, I have to find out where the file stars on disk.

> sudo filefrag -v /swapfile

```
Filesystem type is: ef53
File size of /swapfile is 6442450944 (1572864 blocks of 4096 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..       0:      34816..     34816:      1:            
   1:        1..   30719:      34817..     65535:  30719:             unwritten
   2:    30720..   61439:      65536..     96255:  30720:             unwritten
   3:    61440..   63487:      96256..     98303:   2048:             unwritten
   ...
```

The value that i'm looking for is `34816` for the `resume_offset` parameter.  
`resume` parameter is the partition where the swapfile is located, not the swapfile itself.

Add the needed kernel parameters:
> sudo nano /etc/default/grub  
GRUB_CMDLINE_LINUX_DEFAULT="noquiet nosplash uvcvideo `resume=/dev/md125p3 resume_offset=34816` elevator=noop ...

Now to add the `resume` hook to the initramfs.
>  sudo nano /etc/mkinitcpio.conf  
HOOKS="base udev `resume` autodetect modconf block mdadm_udev filesystems keyboard fsck"

The `resume` hook needs to be after `udev` because the swap partition is referred to with a udev device node.

Finaly apply the changes by rebuilding the initramfs with the mkinitcpio script, as well as re-generate the grub.cfg file:
> sudo mkinitcpio -p linux  
sudo grub-mkconfig -o /boot/grub/grub.cfg

<sub><sup>
References: 
https://wiki.archlinux.org/index.php/Power_management/Suspend_and_hibernate#Hibernation  
https://wiki.archlinux.org/index.php/Mkinitcpio#Image_creation_and_activation  
https://wiki.archlinux.org/index.php/Kernel_parameters#GRUB  
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

Install the CUPS (deamon) and the KDE CUPS utility for KDE4:
> pacman -S cups    
packer -S print-manager-kde4  
sudo systemctl enable org.cups.cupsd.service  
sudo systemctl start org.cups.cupsd.service  

Install EPSON printer drivers:
> sudo packer -S epson-inkjet-printer-escpr

Interfaces:
> Printers can be managed using the CUPS web-interface on `http://localhost:631/` or using the KDE interface, I will use the KDE interface for the configurations.

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

##### Visual Studio

> packer -S visual-studio-code

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

Add PHP to the Apache configuration file:

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

Add phpmyadmin to the PHP configuration file:

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
<IfModule mod_alias.c>
   Alias /phpmyadmin "/usr/share/webapps/phpMyAdmin" 
</IfModule>

<Directory "/usr/share/webapps/phpMyAdmin">  
    DirectoryIndex index.php  
    AllowOverride All  
    Options FollowSymlinks  
    Require local  
</Directory>  
```

Also, if you wish to restrict phpmyadmin to local access you can use the following config. But remember, if you want to be really safe, I would advise to remove it completely. Also, with this config, you will only be able to access by typing  `127.0.0.1` not `localhost`.

```
<IfModule mod_alias.c>
   Alias /phpmyadmin "/usr/share/webapps/phpMyAdmin"
</IfModule>

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

And finnaly include it in the Apache config file:

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

**Check if this configuration file is overriding open_basedir, with "php_admin_value open_basedir" if so, comment this line. I had problems with the inclusion of an external storage because I was giving permissions on the php.ini file (where this config should be) and because of this, it was not working.**

And include it at the bottom of `/etc/httpd/conf/httpd.conf`:

> Include conf/extra/owncloud.conf

Now the easy way to set the correct permissions is to copy and run this script. Replace the `ocpath` variable with the path to your ownCloud directory, and replace the `htuser` variable with your own HTTP user. If the `data` directory does not yet exist, please create it first. (Script extracted from owncloud wiki, modified for Arch)

```
#!/bin/bash
ocpath='/usr/share/webapps/owncloud'
htuser='http'
htgroup='http'

find ${ocpath}/ -type f -print0 | xargs -0 chmod 0640
find ${ocpath}/ -type d -print0 | xargs -0 chmod 0750

chown -R root:${htgroup} ${ocpath}/
chown -R ${htuser}:${htgroup} ${ocpath}/apps/
chown -R ${htuser}:${htgroup} ${ocpath}/config/
chown -R ${htuser}:${htgroup} ${ocpath}/data/
chown -R ${htuser}:${htgroup} ${ocpath}/themes/
chown root:${htgroup} ${ocpath}/.htaccess
chown root:${htgroup} ${ocpath}/data/.htaccess

chmod 0644 ${ocpath}/.htaccess
chmod 0644 ${ocpath}/data/.htaccess
```

Now we just need to activate xcache, for that remove the comment on xcache.ini

> sudo nano /etc/php/conf.d/xcache.ini

And add the entry on config.php:

> sudo nano /etc/webapps/owncloud/config/config.php  
Add the line: 'memcache.local' => '\\OC\\Memcache\\XCache'  
Add the line: xcache.admin.enable_auth=off  

The `xcache.admin.enable_auth=off` solves the `XCache opcode cache will not be cleared because xcache.admin.enable_auth is enabled.` warning.

Finnaly we have to give owncloud access to `/dev/urandom`.
To do so, just attach `:/dev/urandom` (no slash at the end) to open_basedir in php.ini.
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
Include conf/extra/httpd-vhosts.conf  

Now lets setup a virtualhost, but first, we need to comment the existing one in (this can also be done with the purpose of stopping owncloud from taking control of the port 80):

> sudo nano /etc/httpd/conf/extra/owncloud.conf

```
#<VirtualHost *:80>
#    ...
#</VirtualHost>
```
The virtualhost for HTTPS could be re-defined in the `owncloud.conf` file, but I prefer to only have defined application specific configurations, like aliases, in these files. I prefer to have all virtualhosts defined in the file created for that purpose, the `httpd-vhosts.conf`. Also, we could use the `httpd-ssl.conf` because this is a HTTPS virtualhost, but this file has alot of comments that are good for guide lines, but it will create alot of entropy.

Add a new virtualhost that uses HTTPS:

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

Now make sure to listen to the 443 port.
> sudo nano /etc/httpd/conf/httpd.conf  
Add: Listen 443

If you want to map to another port, just change the virtualHost port and the listen port.

For an easier access, we can redirect the port 80 to the port 443 with the following host:
> sudo nano /etc/httpd/conf/extra/httpd-vhosts.conf`:

```
<VirtualHost *:80>
   ServerName OwnCloud
   Redirect permanent / https://LOCAL.OR.PUBLIC.IP:80/
</VirtualHost>
```

Dont forget to forward these TCP ports on your router and local firewall to enable public access.  

If you need to access an external drive, make sure to add the folder to open_basedir and give write permissions to the 'http' user. If the files are not being refreshed, you can force it ba going to phpmyadmin and truncate the tables:

```
TRUNCATE oc_filecache;
TRUNCATE oc_storages;
```

The log in the GUI interface is locaded in: `/usr/share/webapps/owncloud/data/owncloud.log`

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

Avoid entering the password:
> EDITOR=nano visudo  
Add at the end:  
filipe ALL=NOPASSWD: /usr/bin/truecrypt  

Create the mount point:
> mkdir /media/Trabalho

Auto-mount and dismount using systemd:

> sudo nano /usr/lib/systemd/system/truecrypt.service

```
[Unit]
Description=Truecrypt auto mount and dismount
DefaultDependencies=no
RequiresMountsFor=/media/Dados/
Before=shutdown.target reboot.target halt.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/sudo -u filipe /usr/bin/truecrypt -t PATH-TO-VOLUME /media/Trabalho -p 'PASSWORD' -k '' --protect-hidden=no --fs-options=iocharset=utf8
ExecStop=/usr/bin/sudo -u filipe /usr/bin/truecrypt -d

[Install]
WantedBy=multi-user.target
```

If you need, a bash script can be specified instead of the actual command. (See below how.)

`Alternatively` you can use KDE, but it will not unmount properly, if you shutdown using the console.

Auto-Mount the truecrypt container on boot using KDE:  

> System configurations > Start and stop > Add program > System menu and select TrueCrypt.  
nano /home/filipe/.config/autostart/truecrypt.desktop  

```
Exec=truecrypt -t /PATH-TO-VOLUME /media/Trabalho -p 'PASSWORD' -k '' --protect-hidden=no --mount-options=readonly --fs-options=iocharset=utf8  
Terminal=true
```

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

##### Lutris

Nice game launcher/installer

> packer -S lutris

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

##### Network Statistics

> sudo pacman -S vnstat

Query the network traffic: 
> vnstat -q

Viewing live network traffic usage: 
> vnstat -l

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

> sudo nano /usr/lib/systemd/system/awesomescript.service

```
[Unit]
Description=Execute awesomescript on shutdown
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/true
ExecStop=/usr/bin/awesomescript

[Install]
WantedBy=multi-user.target
```

Now create your script:

> sudo nano /usr/bin/awesomescript

```
#!/bin/bash
YOUR COMMANDS
(If you only need to run one command, you can type it directly on the service, just remember to use absolute paths.)
```

Change permissions:
> sudo chmod 755 /usr/bin/awesomescript

To test the script:
> sudo systemctl status awesomescript
sudo systemctl start awesomescript

Since this script is only aimed to work uppon exit, use restart to test:
> sudo systemctl restart awesomescript

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
