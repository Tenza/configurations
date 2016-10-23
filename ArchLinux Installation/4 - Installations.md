# Installations

***DO NOT USE THESE NOTES BLINDLY.***  
***SOME CONFIGURATIONS ARE PERSONAL AND PROBABLY OUTDATED.***

This file is simply my backlog of installations.  
This document does not contain instructions meant as a guide.

### Essential Installations

#### KDE5 Desktop Environment

Install `KDE5`, system language and the `SDDM` display manager that better integrates with KDE5.  
Install the official `breeze` theme and packages to give a uniform look to `Qt` and `GTK` applications.  
Install the `VLC` phonon backend, it has the best upstream support compared with GStreamer, although both can be installed.   Install `libx264` instead of `libx264-10bit` because there are some compatibility issues with 10bit encoders.  

If the packages are not installed on a **single call** to pacman, packages explicitly set in here will be prompted.

<pre>
pacman -S plasma-meta kde-l10n-pt sddm sddm-kcm
pacman -S breeze breeze-kde4 breeze-gtk kde-gtk-config 
pacman -S phonon-qt5 phonon-qt5-vlc libx264
systemctl enable sddm.service
</pre>

To better integrate SDDM with Plasma, it is recommended to edit `/etc/sddm.conf` to use the `breeze` theme.  
Edit this file manually before the first boot, there are some difficulties starting KDE with the SDDM `maui` default theme.

<pre>
sddm --example-config > /etc/sddm.conf
sudo nano /etc/sddm.conf
  [Theme]
  Current=breeze
  CursorTheme=breeze_cursors
  FacesDir=/usr/share/sddm/faces
  ThemeDir=/usr/share/sddm/themes
</pre>

> If the `sddm --example-config` command refuses to run, try `su - root`.

For reference, this can be done using a graphical interface inside KDE, because the package `sddm-kcm` was already installed. This setting is in `Setting > Startup and Shutdown > Login Screen. (2nd tab)`.

<pre>
reboot
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/KDE  
https://wiki.archlinux.org/index.php/desktop_environment  
https://wiki.archlinux.org/index.php/window_manager  
http://forum.doom9.org/showthread.php?t=167654
</sup></sub>

#### Install a terminal

The `plasma-meta` package comes with the bare-bones of the KDE desktop environment. To install more applications without a terminal, simply switch the TTY by pressing `CTRL+ALT+F1-7`, KDE should be running on the `TTY1`.  
To immediately avoid this, consider installing a terminal, for example, `kconsole`.

<pre>
CTRL+ALT+F2
pacman -S kconsole
</pre>

> Note that the default TTY was changed in the migration of `systemd/logind` on October 2012 to [fix a problem](https://bugs.archlinux.org/task/32206). This is different from KDE4, that used to run on the TTY7.

#### Activate NetworkManager

The `NetworkManager` package is responsible for managing the network functionalities, and it was already installed as a dependency to the KDE `plasma-nm` front-end. Simply `status/start/enable` the service if needed. Also, if the `dhcpcd` or `netctl-auto` services were enabled, they need to be [stopped and disabled](https://github.com/Tenza/configurations/blob/master/ArchLinux%20Installation/2%20-%20Base%20System%20Configuration.md#optional-not-recommended-start-network-connections-at-boot) first to avoid conflicts.

<pre>
systemctl enable NetworkManager.service
</pre>

> Make sure that your WiFi/Bluetooth card is active (typically a keyboard shortcut on a laptop) in order to make the adapter visible to the kernel.

NetworkManager by default stores passwords in clear text in the connection files at `/etc/NetworkManager/system-connections/`. It is preferable to save the passwords in encrypted form instead of clear text, this can be achieved by storing them in a keyring which NetworkManager then queries for the passwords. For KDE, the keyring daemon is [KDE Wallet](https://github.com/Tenza/configurations/blob/master/ArchLinux%20Installation/4%20-%20Installations.md#kwallet).

#### Activate Bluetooth

The `bluez` package is responsible for managing the bluetooth connections, and it was already installed as a dependency to the KDE `bluedevil` front-end. Simply `status/start/enable` the service if needed.

<pre>
systemctl enable bluetooth.service
</pre>

> Make sure that your WiFi/Bluetooth card is active (typically a keyboard shortcut on a laptop) in order to make the adapter visible to the kernel.

If there is an error regarding the `sap-driver` when querying the status of the bluetooth service, know that this behaviour is expected, and can be solved by simply adding the `--noplugin=sap` to the service `ExecStart`.

<pre>
nano /usr/lib/systemd/system/bluetooth.service
  ExecStart=/usr/lib/bluetooth/bluetoothd --noplugin=sap
</pre>

#### KWallet

The `kwallet` and `kwallet-pam` packages should already be installed as a dependency to the `kio` and `plasma-meta` packages. Once the network connects, or any other service requires the keyring daemon, a configuration window should pop-up, and a wallet should be created. 

There are two encryption methods available in KWallet, the standard Blowfish and GnuPG a free implementation of the OpenPGP standard as defined by RFC4880 (also known as PGP). The GnuPG is clearly superior, even more so, when it is taken into consideration that the Blowfish implementation in KWallet [is absoluty terrible](http://security.stackexchange.com/questions/43988/security-of-the-kwallet-password-encrypting-application). The downside of using GnuPG is that `kwallet-pam` is not compatible with it, so it is not yet possible to automatically unlock the wallet at boot.

Also, consider installing the `kwalletmanager` in order to manage multiple wallets. This can be useful to manage multiple wallets using the two different types of encryption available.

<pre>
pacman -S kwalletmanager
</pre>

###### GnuPG

To use GnuPG encryption, a key-pair has to be created first. The `gnupg` package was already installed as a dependency of `archboot` and it will enable the creation of the keys. Also make sure the `pinentry` package is installed because it will be used to create the dialog which GnuPG uses for passphrase entry. First, the program that will be used to prompt the passphrase has to be defined, than `gnupg` can be called **with the logged user** to generate the keys.

<pre>
nano ~/.gnupg/gpg-agent.conf
  pinentry-program /usr/bin/pinentry-qt
  
gpg-connect-agent reloadagent /bye
gpg --full-gen-key
</pre>

###### Blowfish with automatic unlock

To use Blowfish encryption, simply select the option on the wallet creation window, there isn't any additional configuration needed. The advantage that Blowfish has over `GnuPG` is the support it has from `kwallet-pam`, in order to allow automatic unlock at boot. 

KWallet will prompt for the password every first time a application that integrates with `kwallet` requires it. This can happen for example, at every startup, when the NetworkManager queries for the password of a wireless network. 

To prevent this behaviour, the wallet has to be called `kdewallet` and the password chosen has to be the same as your username. Additionally set the PAM configuration of SDDM.

<pre>
nano /etc/pam.d/sddm
  auth            include         system-login
  <b>auth            optional        pam_kwallet5.so</b>
  account         include         system-login
  password        include         system-login
  session         include         system-login
  <b>session         optional        pam_kwallet5.so  auto_start</b>
</pre>

#### Gnome-keyring

Some applications use the gnome-keyring to store credentials instead of KWallet. Install the `gnome-keyring` as well as `seahorse` in order to manage the credentials. 

<pre>
pacman -S gnome-keyring seahorse
</pre>

###### Automatic unlock

Like Kwallet, there are a few conditions to automatically unlock the wallet using PAM. The password must be be the same as your username, and additionally set the PAM configuration of SDDM.

<pre>
nano /etc/pam.d/sddm
  auth            include         system-login
  auth            optional        pam_kwallet5.so
  <b>auth            optional        pam_gnome_keyring.so</b>
  account         include         system-login
  password        include         system-login
  session         include         system-login
  session         optional        pam_kwallet5.so         auto_start
  <b>session         optional        pam_gnome_keyring.so    auto_start</b>
</pre>

By default a new wallet is created when the gnome-keyring is automatically unlocked. In order to save the credentials to this wallet, it must be set as the default wallet, use seahorse to do this.

#### Fstab

To mount additional partitions or disks at boot use the `fstab` file. Start by listing the device blocks with the filesystems flag, in order to get the UUID of the partition to be mounted. Than install the needed packages to give support to the partition filesystem type. Lastly make the dir for that mount-point, and set the entry in the `fstab` file with the information gathered previously.

<pre>
lsblk -f
pacman -S ntfs-3g
mkdir /Dados
nano /etc/fstab
  # /dev/md125p4 LABEL=Dados
  UUID=466C899D6C8987FF            /dados          ntfs-3g         defaults,permissions,nofail       0 0
</pre>

| Switch | Description | 
| --- | --- | 
| permissions | Sets the permissions, to the same permissions set on the files and dirs. This will enable the user to set permissions of the files and dirs using for example `chown` or `chmod`. | 
| uid=1000,dmask=027,fmask=137 | These parameters can be used to explicitly set the permissions of all files and dirs. This will block the user from changing the permissions of the files and dirs. |
| nofail | Even if this mount fails at boot, it will not stop the computer from booting. | 

> Setting `ntfs` or `ntfs-3g` is basically the same. Check with `ls /sbin/mount.ntfs* -l`

#### Fonts

The following set of fonts are UTF8 fonts, they are universal and should be generally useful for the average user.

<pre>
pacman -S ttf-dejavu ttf-freefont ttf-liberation ttf-droid ttf-inconsolata
</pre>

> I use the `ttf-inconsolata` as my terminal font, this font is also very good for programming purposes.

#### Power Management

Power Management consists of two main parts, configuration of the Linux kernel, which interacts with the hardware and configuration of userspace tools, which interact with the kernel and react to its events.  

###### Power Management with Kernel Parameters

The `i915` kernel module of the Intel graphics, allows for configuration via kernel parameters. Some of the module options impact power saving. List all the options, and set the kernel parameters. This set of options should be generally safe to enable.

<pre>
modinfo -p i915
nano /etc/default/grub
  GRUB_CMDLINE_LINUX_DEFAULT="noquiet nosplash i915.enable_rc6=1 i915.enable_fbc=1 i915.semaphores=1 915.lvds_downclock=1"

grub-mkconfig -o /boot/grub/grub.cfg 
</pre>

| Switch | Description | 
| --- | --- | 
| i915.enable_rc6=1 | The Intel i915 RC6 feature allows the Graphics Processing Unit (GPU) to enter a lower power state during GPU idle. The i915 RC6 feature applies to Intel Sandybridge and later processors. | 
| i915.enable_fbc=1 | Framebuffer compression reduces the memory bandwidth on screen refreshes and depending on the image in the framebuffer can reduce power consumption. This option is not enabled by default as on some hardware framebuffer compression causes artifacts when repainting when using composites. Some users report this breaks when using Unity 3D.  | 
| i915.semaphores=1 | Use semaphores for inter-ring sync. |
| 915.lvds_downclock=1 | This kernel option will down-clock the LVDS refresh rate, and this in theory will save power. For systems that do not support LVDS down-clocking the screen can flicker. |

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Power_management  
https://wiki.ubuntu.com/Kernel/PowerManagement/PowerSavingTweaks  
https://wiki.archlinux.org/index.php/intel_graphics#Module-based_Powersaving_Options  
</sup></sub>

###### Power Management with Userspace tools

There are multiple tools within userspace to enable additional power management. Only run one of these tools to avoid possible conflicts as they all work more or less in a similar way. Laptop Mode Tools (LMT) is the utility that is going to be configured, it is considered by many to be the de-facto utility for power management.

<pre>
pacaur -S laptop-mode-tools-git acpid bluez-utils wireless_tools ethtool
systemctl enable laptop-mode.service
systemctl enable acpid.service
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Power_management  
https://wiki.archlinux.org/index.php/Laptop_Mode_Tools  
https://wiki.archlinux.org/index.php/acpid  
</sup></sub>

### Installations with configurations

#### Skype

Skype no longer needs to be in complete lockdown mode, because there is a web version available. With this feature, a few front-end applications are now available. I'm using the official version, that is simply a wrapper of the Skype WebRTC for Linux.

<pre>
pacaur -S skypeforlinux-bin
</pre>

> Skype uses the gnome-keyring to store the credentials. Make sure it is properly configured.

###### Skype startup in tray

Skype is able to `Close to tray`, but it always starts with the main window visible. Closing to tray at startup is not yet available, and command line parameters like `--silent` are also not available. To force this behavior, I created a simple script using `xdotool` to close the main window of skype at startup.

<pre>
pacman -S xdotool
nano /home/filipe/Scripts/Skype
  #!/bin/bash
  
  sleep 5
  
  /usr/bin/skypeforlinux
  
  sleep 1
  
  # Close window
  if [[ -n `pidof skypeforlinux` ]];then
      WID=`xdotool search --name "Skype for Linux Alpha"`
      xdotool windowactivate --sync $WID
      xdotool key --clearmodifiers ALT+F4
  fi

chmod 755 /home/filipe/Scripts/Skype
(Add the script to startup (symlink) with KDE)
</pre>

<sub><sup>
References:  
http://unix.stackexchange.com/questions/85205/is-there-a-way-to-simulate-a-close-event-on-various-windows-using-the-terminal  http://how-to.wikia.com/wiki/How_to_gracefully_kill_(close)_programs_and_processes_via_command_line
</sup></sub>

#### Steam

Steam is not officially supported in ArchLinux, as such some fixes are needed to get things functioning properly.

<pre>
pacman -S steam
</pre>

> On 64bit systems, make sure multilib is enabled, and the 32bit variants of the graphics and sound drivers are installed.

##### Steam Runtime

One of the most common errors is due to broken/missing libraries, and steam may fail to start. Steam installs its own older versions of some libraries collectively called the "Steam Runtime". These will often conflict with the libraries included in Arch Linux.

<pre>
libGL error: unable to load driver: i965_dri.so
libGL error: driver pointer missing
libGL error: failed to load driver: i965
libGL error: unable to load driver: i965_dri.so
libGL error: driver pointer missing
libGL error: failed to load driver: i965
libGL error: unable to load driver: swrast_dri.so
libGL error: failed to load driver: swrast
</pre>

There are two ways to fix this, forcing Steam to load the up-to-date system libraries with a dynamic linker, or to force Steam to use only the system libraries. I will use the dynamic linker method for compatibility reasons.

To use the dynamic linker, the variable `LD_PRELOAD` has to be set before calling steam. Setting `LD_PRELOAD` to the path of a shared object, ensures that the passed files will be loaded before any other library, including the C runtime. 

<pre>
<b>LD_PRELOAD='/usr/$LIB/libstdc++.so.6 /usr/$LIB/libgcc_s.so.1 /usr/$LIB/libxcb.so.1 /usr/$LIB/libgpg-error.so'</b> /usr/bin/steam %U
</pre>

To use the native runtime, simply set the variable `STEAM_RUNTIME=0`, keep in mind that this method possibly requires the installation of additional 32bit libraries, if you are missing any libraries from the Steam runtime.

<pre>
<b>STEAM_RUNTIME=0</b> /usr/bin/steam %U
</pre>

##### Close to tray

By default steam closes when the main window is closed. To enable the "close to tray" behavior, the variable `STEAM_FRAME_FORCE_CLOSE` has to be set.

<pre>
LD_PRELOAD='/usr/$LIB/libstdc++.so.6 /usr/$LIB/libgcc_s.so.1 /usr/$LIB/libxcb.so.1 /usr/$LIB/libgpg-error.so' <b>STEAM_FRAME_FORCE_CLOSE=1</b> /usr/bin/steam %U
</pre>

##### Start silently

By default steam opens the main window when it starts. This can be inconvenient if steam is set to start at boot. To disable this behavior, and hide the main window the `-silent` parameters has to be passed.

<pre>
LD_PRELOAD='/usr/$LIB/libstdc++.so.6 /usr/$LIB/libgcc_s.so.1 /usr/$LIB/libxcb.so.1 /usr/$LIB/libgpg-error.so' STEAM_FRAME_FORCE_CLOSE=1 /usr/bin/steam %U <b>-silent</b>
</pre>

##### Tray icon

Steam default tray icon cannot be seen very well with a white taskbar, to replace it with the default icon use the following commands. First backup the current mono icon, and then copy the default icon with the same name.

<pre>
sudo mv /usr/share/pixmaps/steam_tray_mono.png  /usr/share/pixmaps/steam_tray_mono.png.bak 
sudo cp /usr/share/pixmaps/steam.png /usr/share/pixmaps/steam_tray_mono.png
</pre>

> These change might be lost once steam updates itself.

##### All together

For me the easiest way to have this all toguether is to create a simple script. You can also simply use KDE to edit the `.desktop` file of steam, and that will work just fine, but it will probably not survive updates.

Keep in mind that KDE copies the file `/usr/share/applications/steam.desktop` to your home dir `/home/filipe/.local/share/applications/steam.desktop` with the modifications when editing manually.

<pre>
nano /home/filipe/Scripts/Steam
  #!/bin/bash

  sleep 5

  cp /usr/share/pixmaps/steam.png /usr/share/pixmaps/steam_tray_mono.png
  LD_PRELOAD='/usr/$LIB/libstdc++.so.6 /usr/$LIB/libgcc_s.so.1 /usr/$LIB/libxcb.so.1 /usr/$LIB/libgpg-error.so' STEAM_FRAME_FORCE_CLOSE=1 /usr/bin/steam %U -silent
  
chmod 755 /home/filipe/Scripts/Skype
(Add the script to startup (symlink) with KDE)
</pre>

### Simple Installations

#### File Manager

<pre>
pacman -S dolphin
</pre>

#### Advanced text editor

<pre>
pacman -S kate
</pre>

#### Archiving Tool

<pre>
pacman -S ark p7zip zip unzip unrar
</pre>

#### Browser

<pre>
pacman -S firefox firefox-i18n-pt-pt
</pre>

#### Qt

<pre>
pacman -S qt5-base qt5-doc qt5-tools qt5-examples qtcreator
pacman -S cmake gdb valgrind
</pre>

#### Chat client

<pre>
pacman -S pidgin pidgin-otr
</pre>

> When adding a XMPP account the "Resource" field is optional, is used to define your instance if you have several locations you chat from, for example "Home" or "Work".

> Pidgin restores it's state at boot, no need to add to the startup menu.

#### File and folder compare

<pre>
pacman -S meld
</pre>

> Kompare no longer crashes, but is still not as good as meld.

#### Torrent

<pre>
pacman -S qbittorrent
</pre>

> QBittorrent restores it's state at boot, no need to add to the startup menu.

#### Office

<pre>
pacman -S libreoffice-fresh libreoffice-fresh-pt
pacman -S hunspell-en 
paraur -S hunspell-pt_pt
</pre>

> I prefer Libreoffice over Calligra due to the compatibility and similarity to MS office.

#### Image Editor

<pre>
paraur -S gimp
</pre>

> WARNING: GIMP will probably override your Image Viewer by default.

#### Image Viewer

<pre>
paraur -S photoqt
</pre>

> PhotoQt will probably be overriden by GIMP if installed first, so consider installing GIMP first.
> I prefer photoqt over gwenview due to its simplicity.

### Aditional configurations

#### PulseAudio Audiophile

By default, PulseAudio (PA) uses very conservative settings. This will work fine for most audio media as you will most likely have 44,100Hz sample rate files. However, if you have higher sample rate recordings it is recommended that you increase the sample rate that PA uses.

<pre>
nano /etc/pulse/daemon.conf
  default-sample-format = s32le 
  default-sample-rate = 96000 
  resample-method = speex-float-5 
</pre>

For the most genuine resampling at the cost of high CPU usage (even on 2011 CPUs) set `resample-method`.

<pre>
nano /etc/pulse/daemon.conf
  resample-method = src-sinc-best-quality 
</pre>

To change the channels set by pulseaudio (they might be wrong), set `default-sample-channels`.

<pre>
nano /etc/pulse/daemon.conf
  default-sample-channels = 3
  default-channel-map = front-left,front-right,lfe
</pre>

<sub><sup>
References: 
http://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Audiophile/
</sup></sub>

### Problem Solving

#### Missing firmware modules

When initramfs is being rebuild after a kernel update, some kernel warnings may appear.

<pre>
==> WARNING: Possibly missing firmware for module: wd719x
==> WARNING: Possibly missing firmware for module: aic94xx
</pre>

Simply install the following modules to fix the warnings.

<pre>
pacaur -S wd719x-firmware
pacaur -S aic94xx-firmware
</pre>

#### Libreoffice does not autocorrect

Check if the checkbox has you a blue tick or a little `A` next to the language, if it does, the language pack is installed, simply select it to start showing in the language bar bellow. If it doesn't, install the appropriate `hunspell` language pack.

<pre>
Tools -> Options -> Language Settings -> Languages -> Default language for document
</pre>

### Look & feel

Just some personal configurations of look and feel.

<pre>
Configurate tray > Configure visibility.
Configurate taskbar > No groups, manually ordered.
Configurate system configurations > Classic tree, remove detailed tips, expand first level.
Configurate startmenu > Change the icon to <a href="http://gabriela2400.deviantart.com/art/Arch-Linux-Start-Icons-175557586">one of these</a>.
Dolphin > Adjust window properties:  Show in details, show hidden files. Apply to all folders.
Dolphin > Additional Information > Name, Size, Date, Type, Location, permissions, owner.
Dolphin > Change default start up, show location bar, show filter.
Dolphin > Right click on a drive > Icon size > Large.
Dolphin > Right click on top menu > Icon size > Large.
Dolphin > Icon size 32px (drag lower bar)
System configurations > Doble-click to open.
System configurations > Add account image.
System configurations > Screen borders > Lower right, Show Screen.
System configurations > Gobal hotkeys > KDE Sessions > Lock Screeb with Windows+L.
VLC > Enable multiple instances.
</pre>

#### SDDM Breeze UserImage

Set a user account image using KDE, this will enable the Breeze SDDM theme to display it.
This should create a `.face` file in the home dir as well as a symbolic link named `.face.icon`.

<pre>
Configurate system > Account Details > User Manager
</pre>

SDDM might not be able to read this image at startup, so the image has to also be copied to `/usr/share/sddm/faces/` with the name of the user account, for example `filipe.face.icon`.

<pre>
cp /my/user/avatar.png /usr/share/sddm/faces/filipe.face.icon
chmod 644 /usr/share/sddm/faces/filipe.face.icon
</pre>

#### SDDM Breeze Background

Personally I like to have the same background for Startup and LockScreen (and wallpaper).
To change the LockScreen background use KDE System settings.

<pre>
Configurate system > Desktop > Lock screen > Background
</pre>

To change the Startup background should also be possible to use KDE.
But if for some reason it fails to do so, it is possible to do it manually.

<pre>
cp /my/user/background.png /usr/share/sddm/themes/breeze/background.png
chmod 644 /usr/share/sddm/themes/breeze/background.png
nano /usr/share/sddm/themes/breeze/theme.conf
  [General]
  background=background.jpg
</pre>

#### Bootloader GRUB2 Theme

My theme of choice when it comes to GRUB is [this one](https://github.com/Generator/Grub2-themes).
Start by downloading and copy the files to the GRUB2 location.
<pre>
git clone git://github.com/Generator/Grub2-themes.git
cp -r Grub2-themes/{Archlinux,Archxion} /boot/grub/themes/
</pre>

Edit the grub configuration file, and rebuild the GRUB menu.

<pre>
nano /etc/default/grub
  #GRUB_THEME="/path/to/gfxtheme"
    GRUB_THEME="/boot/grub/themes/Archxion/theme.txt" 
    GRUB_THEME="/boot/grub/themes/Archlinux/theme.txt"
  
  GRUB_GFXMODE=auto 
    GRUB_GFXMODE=1024x768
  
grub-mkconfig -o /boot/grub/grub.cfg
</pre>
