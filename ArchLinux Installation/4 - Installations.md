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
pacman -S phonon-qt5 phonon-qt5-vlc phonon-qt4-vlc libx264
systemctl enable sddm.service
</pre>

To better integrate SDDM with Plasma, it is recommended to edit `/etc/sddm.conf` to use the `breeze` theme.  
Edit this file manually before the first boot, there are some difficulties starting KDE with the SDDM `maui` default theme.

<pre>
sddm --example-config > /etc/sddm.conf
nano /etc/sddm.conf
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

If there is an error regarding the `sap-driver` when querying the status of the bluetooth service, know that this behavior is expected, and can be solved by simply adding the `--noplugin=sap` to the service `ExecStart`.

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
mkdir /dados
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
  GRUB_CMDLINE_LINUX_DEFAULT="i915.enable_rc6=1 i915.enable_fbc=1 i915.semaphores=1 915.lvds_downclock=1"

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

#### Hibernate

To activate the hibernate functionality, some kernel parameters have to be added, as well as the `resume` hook to initramfs. The parameters values differ depending on the configuration used. For example if using a partition/file and with/without encryption.

Start by adding the `resume` hook to the initramfs.
This hook needs to be after `udev` because the swap is referred to with a udev device node.

<pre>
nano /etc/mkinitcpio.conf  
  HOOKS="base udev `resume` autodetect modconf block mdadm_udev filesystems keyboard fsck"
</pre>

##### Using a Swapfile

When using a swap file instead of a partition, it has to first be determined where the file starts on disk.

<pre>
filefrag -v /swapfile

Filesystem type is: ef53
File size of /swapfile is 6442450944 (1572864 blocks of 4096 bytes)
 ext:     logical_offset:        physical_offset: length:   expected: flags:
   0:        0..       0:      34816..     34816:      1:            
   1:        1..   30719:      34817..     65535:  30719:             unwritten
   2:    30720..   61439:      65536..     96255:  30720:             unwritten
   3:    61440..   63487:      96256..     98303:   2048:             unwritten
   ...
</pre>

The needed value, in this case, is `34816`. It will be used by the `resume_offset` kernel parameter.  

###### Without Encryption

The `resume` kernel parameter is the `partition` where the swapfile is located, not the swapfile itself.  
The `resume_offset` kernel parameter can be omitted if using a swap partition.

<pre>
nano /etc/default/grub  
  GRUB_CMDLINE_LINUX_DEFAULT="resume=/dev/md125p3 resume_offset=34816"
</pre>

###### With Encryption

The `resume` kernel parameter is the `device mapper` where the swapfile is located, not the swapfile itself.  
The `resume_offset` kernel parameter can be omitted if using a swap partition.

<pre>
nano /etc/default/grub  
  GRUB_CMDLINE_LINUX_DEFAULT="resume=/dev/mapper/ArchCrypt resume_offset=34816"
</pre>

##### Apply Changes

Apply the changes by rebuilding the initramfs with the mkinitcpio script, and regenerate the grub.cfg file.

<pre>
mkinitcpio -p linux  
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Power_management/Suspend_and_hibernate#Hibernation  
https://wiki.archlinux.org/index.php/Mkinitcpio#Image_creation_and_activation  
https://wiki.archlinux.org/index.php/Kernel_parameters#GRUB  
</sup></sub>

### Installations with configurations

#### Skype

Skype no longer needs to be in complete lockdown mode, because there is a web version available. With this feature, a few front-end applications are now available. I'm using the official version, that is simply a wrapper of the Skype WebRTC for Linux.

<pre>
pacaur -S skypeforlinux-bin
</pre>

> Skype uses the gnome-keyring to store the credentials. Make sure it is properly configured.

###### Skype startup in tray

Skype is able to `Close to tray`, but it always starts with the main window visible. Closing to tray at startup is not yet available, and command line parameters like `--silent` are also not available. To force this behavior, I created a simple script using `xdotool` to close the main window of skype at startup. The script also checks for the network status before opening, because sometimes skype displays that it wasn't able to connect, (and it will not reconnect) if the network connection isn't already established when it opens.

<pre>
pacman -S xdotool
  nano /home/filipe/Scripts/Skype
  #!/bin/bash

  #Check connectivity
  while true
  do
      if ping -w 1 -c 1 google.com >> /dev/null 2>&1; then
          echo "Online"
          break
      else
          echo "Offline"
          sleep 10
      fi
  done

  #Start Skype and sleep 1 second so xdotool can take effect
  /usr/bin/skypeforlinux
  sleep 1

  # Close window
  if [[ -n `pidof skypeforlinux` ]];then
      WID=`xdotool search --name "Skype for Linux Alpha"`
      xdotool windowactivate --sync $WID
      xdotool key --clearmodifiers ALT+F4
  fi

chmod 711 /home/filipe/Scripts/Skype
(Add the script to startup (symlink) with KDE)
</pre>

<sub><sup>
References:  
http://unix.stackexchange.com/questions/85205/is-there-a-way-to-simulate-a-close-event-on-various-windows-using-the-terminal   http://how-to.wikia.com/wiki/How_to_gracefully_kill_(close)_programs_and_processes_via_command_line
</sup></sub>

#### Steam

Steam is not officially supported in ArchLinux, as such some fixes are needed to get things functioning properly.

<pre>
pacman -S steam
</pre>

> On 64bit systems, make sure multilib is enabled, and the 32bit variants of the graphics and sound drivers are installed.

##### Steam Runtime

One of the most common errors is due to broken/missing libraries, and Steam may fail to start. Steam installs its own older versions of some libraries collectively called the "Steam Runtime". These can conflict with the libraries included in Arch Linux.

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

There a few ways to fix this, forcing Steam to load the up-to-date system libraries with a dynamic linker, or to force Steam to use only the system libraries, or to simply use the steam-runtime.

To use the dynamic linker, the variable `LD_PRELOAD` has to be set before calling Steam. Setting `LD_PRELOAD` to the path of a shared object, ensures that the passed files will be loaded before any other library, including the C runtime. 

<pre>
<b>LD_PRELOAD='/usr/$LIB/libstdc++.so.6 /usr/$LIB/libgcc_s.so.1 /usr/$LIB/libxcb.so.1 /usr/$LIB/libgpg-error.so'</b> /usr/bin/steam %U
</pre>

To run natively, simply set the variable `STEAM_RUNTIME=0`, keep in mind that this method possibly requires the installation of additional 32bit libraries, if you are missing any libraries from the Steam runtime.

<pre>
<b>STEAM_RUNTIME=0</b> /usr/bin/steam %U
</pre>

Run steam on it's own runtime, with older, but accurate libriaries. 

<pre>
<b>/usr/bin/steam-runtime %U</b> 
</pre>

##### Close to tray

By default steam closes when the main window is closed. To enable the "close to tray" behavior, the variable `STEAM_FRAME_FORCE_CLOSE` has to be set.

<pre>
<b>STEAM_FRAME_FORCE_CLOSE=1</b> /usr/bin/steam-runtime %U
</pre>

##### Start silently

By default steam opens the main window when it starts. This can be inconvenient if steam is set to start at boot. To disable this behavior, and hide the main window the `-silent` parameters has to be passed.

<pre>
STEAM_FRAME_FORCE_CLOSE=1 /usr/bin/steam-runtime %U <b>-silent</b>
</pre>

##### Tray icon

Steam default tray icon cannot be seen very well with a white taskbar, to replace it with the default icon use the following commands. First backup the current mono icon, and then copy the default icon with the same name.

<pre>
mv /usr/share/pixmaps/steam_tray_mono.png  /usr/share/pixmaps/steam_tray_mono_bak.png 
cp /usr/share/pixmaps/steam.png /usr/share/pixmaps/steam_tray_mono.png
</pre>

> These change might be lost once steam updates itself.

##### All together

For me the easiest way to have this all together is to create a simple script. It is possible to simply edit the `.desktop` file of steam, but it will probably not survive updates. Additionally, this script will only start steam when the network is active.

Keep in mind that KDE copies the file `/usr/share/applications/steam.desktop` to your home dir `/home/filipe/.local/share/applications/steam.desktop` with the modifications when editing manually.

<pre>
nano /home/filipe/Scripts/Steam
  #!/bin/bash

  #Check connectivity
  while true
  do
    if ping -w 1 -c 1 google.com >> /dev/null 2>&1; then
      echo "Online"
      break
    else
      echo "Offline"
      sleep 10
    fi
  done

  #Replace icon file to survive steam updates
  cp /usr/share/pixmaps/steam.png /usr/share/pixmaps/steam_tray_mono.png

  #Use Steam runtime
  STEAM_FRAME_FORCE_CLOSE=1 /usr/bin/steam-runtime %U -silent

  #Use Steam Native with Dynamic Linker
  #LD_PRELOAD='/usr/$LIB/libstdc++.so.6 /usr/$LIB/libgcc_s.so.1 /usr/$LIB/libxcb.so.1 /usr/$LIB/libgpg-error.so' STEAM_FRAME_FORCE_CLOSE=1 /usr/bin/steam %U -silent
  
chmod 711 /home/filipe/Scripts/Steam
(Add the script to startup (symlink) with KDE)
</pre>

#### Dropbox with CryFS

Dropbox is a file sharing system with a GNU/Linux client.  
Cryfs is a cryptographic filesystem for the cloud, and at the moment is under heavy development. This is a much better alternative than using TrueCrypt containers (that I was previously using), to securely save data on cloud services. Since this is still under development some errors are expected and the data should have a backup.

I've made a script to mount and unmount as well as automatically backup all data. This script requires the `expect` command, a tool that is commonly used for automating interactive applications, and the `rsync` application to backup the changed data.

<pre>
pacaur -S cryfs
pacaur -S dropbox
pacman -S expect
pacman -S rsync
</pre>

##### Auto Mount

The script automates the mount and backup processes, as well as to solve a problem I had with CryFS.
The problem was that CryFS is extremely slow under certain filesystems, like NTFS. It's too slow to be able to browse the directory structure normally with dolphin, because dolphin also reads the files metadata to enable functionalities like file-preview.

With this in mind, I decided that I would work only in my RAW (or backup) folder, and would use the script to mount and synchronize any changes made only at boot. My CryFS mount folder is a hidden folder, and the RSync command uses the `--delete` flag to make sure I don't work under the mount folder, because any changes in there will be lost.

Replace the CryFS password and appropriate directories, create and set the mount script.

<pre>
nano /home/filipe/Scripts/CryFS
  #!/bin/bash

  dir_cloud="/dados/Dropbox/Privado/Trabalho/"
  dir_cryfs="/dados/Trabalho/.CryFS/"
  dir_raw="/dados/Trabalho/Trabalho/"

  log_rsync="/dados/Trabalho/RSync.log"

  password="YOUR_CRYFS_PASSWORD"

  #Check connectivity
  while true
  do
      if ping -w 1 -c 1 google.com >> /dev/null 2>&1; then
          echo "Online"
          break
      else
          echo "Offline"
          sleep 10
      fi
  done

  #Check if CryFS is already mounted
  if ! mount | grep cryfs > /dev/null; then

      printf "Mounting CryFS\n\n"

  #Expect commands enclosed within the CRYFS block, don't indent.
  /usr/bin/expect <<- CRYFS
      spawn cryfs $dir_cloud $dir_cryfs

      expect "*?assword:*"
      send -- "$password\r"

      interact
      expect EOF
  CRYFS

    #Check if was mounted with success, and start RSync Synchronization with logs and delete flag.
      if mount | grep cryfs > /dev/null; then
          printf "\nCryFS Mounted\n"
          printf "\nStarting RSync Synchronization\n"

          rsync -avh --delete --log-file=$log_rsync $dir_raw* $dir_cryfs

          printf "\nRSync Complete\n"
      else
          printf "\nCryFS NOT Mounted\n"
      fi

  else

      printf "CryFS Already Mounted\n\n"

  fi
  
chmod 711 /home/filipe/Scripts/CryFS
(Add the script to startup (symlink) with KDE)
</pre>

> Note that a script that uses expect, normally uses `#!/usr/bin/expect`, and has the `ext` extension. But in this case, bash commands are also needed, so the expect commands are inside a block within the bash script.

##### Auto Unmount

Replace the appropriate directories, create and set the unmount script.

<pre>
nano /home/filipe/Scripts/CryFSUnmount
  #!/bin/bash

  dir_cryfs="/dados/Trabalho/.CryFS/"

  #Check if CryFS is already mounted
  if mount | grep cryfs > /dev/null; then

      printf "Unmounting CryFS\n\n"
      fusermount -u $dir_cryfs
      printf "CryFS Unmounted\n\n"

  else

      printf "CryFS NOT Mounted\n\n"

  fi
  
chmod 711 /home/filipe/Scripts/CryFSUnmount
(Add the script to shutdown (symlink) with KDE)
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

#### Chat client

<pre>
pacman -S pidgin pidgin-otr
</pre>

> When adding a XMPP account the "Resource" field is optional, is used to define your instance if you have several locations you chat from, for example "Home" or "Work".

> Pidgin restores it's state at boot, no need to add to the startup menu.

#### Torrent

<pre>
pacman -S qbittorrent
</pre>

> QBittorrent restores it's state at boot, no need to add to the startup menu.

#### Office

<pre>
pacman -S libreoffice-fresh libreoffice-fresh-pt
pacman -S hunspell-en 
pacaur -S hunspell-pt_pt
</pre>

> I prefer Libreoffice over Calligra due to the compatibility and similarity to MS office.

#### File and folder compare

<pre>
pacman -S meld
</pre>

> Kompare no longer crashes, but is still not as good as meld.

#### Image Editor

<pre>
pacaur -S gimp
</pre>

> WARNING: GIMP will probably override your Image Viewer by default.

#### Image Viewer

<pre>
pacaur -S photoqt
</pre>

> PhotoQt will probably be overridden by GIMP if installed first, so consider installing GIMP first.
> I prefer photoqt over gwenview due to its simplicity.

#### Audio Player

<pre>
pacman -S clementine gst-plugins-base gst-plugins-good gst-libav
</pre>

> Make sure that the gstreamer plugins are installed and updated, otherwise the error "Your GStreamer installation is missing a plug-in" will be displayed.

> Clementine restores it's state at boot, no need to add to the startup menu.

#### Qt

<pre>
pacman -S qt5-base qt5-doc qt5-tools qt5-examples qtcreator
pacman -S cmake gdb valgrind
</pre>

#### Teamviewer

<pre>
paraur -S teamviewer
systemctl start teamviewerd.service
</pre>

#### Calculators

<pre>
pacman -S kcalc
pacaur -S extcalc
</pre>

#### ClamAV Antivirus

<pre>
pacman -S clamav
freshclam
systemctl enable freshclamd.service
systemctl enable clamd.service
</pre>

#### UFW Firewall

<pre>
pacman -S ufw
ufw enable
systemctl enable ufw.service
</pre>

#### KDE Installs

<pre>
pacman -S okular
pacman -S spectacle
</pre>

### Additional configurations

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

#### Change default DNS

The DNS can be set in the `/etc/resolv.conf` file, and this can hold up to three `nameserver` entries. Before setting the DNS, the NetworkManager has to be stopped and configured to not overwrite this file, otherwise the changes will be lost.

<pre>
systemctl stop NetworkManager.service
nano /etc/NetworkManager/NetworkManager.conf
  dns=none
</pre>

##### Without DNSCrypt

To simply set DNS resolvers, without any authentication of the DNS traffic between user and DNS resolver.

<pre>
nano /etc/resolv.conf
  nameserver dns.ip.address
  nameserver dns.ip.address
  
systemctl start NetworkManager.service 
</pre>

##### With DNSCrypt

DNSCrypt encrypts and authenticates DNS traffic between user and DNS resolver. It prevents local spoofing of DNS queries, ensuring DNS responses are sent by the server of choice.

Select one of the resolvers provided by the [upstream](https://github.com/jedisct1/dnscrypt-proxy/blob/master/dnscrypt-resolvers.csv) and replace the 'resolver name' with the selected name.

<pre>
pacman -S dnscrypt-proxy

nano /etc/resolv.conf
  nameserver 127.0.0.1
  
nano /usr/lib/systemd/system/dnscrypt-proxy.service 
  ExecStart=/usr/bin/dnscrypt-proxy --resolver-name='resolver name'
  
systemctl enable dnscrypt-proxy.service
reboot
</pre>

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

#### Bootloader GRUB2 Locale Error Message

If GRUB flashes a message for a split second at boot, it is possible that the message is like the following:
<pre>
Welcome to Grub!
error: file '/boot/grub/locale/pt.mo' not found
</pre>

This can happen because the configured locale does not exist. To fix this, simply copy one of the existing configs to the missing one, and the locale file will now be read without problems.
<pre>
cp /boot/grub/locale/pt_BR.mo /boot/grub/locale/pt.mo
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
GIMP > Single window mode.
</pre>

#### Pacman

Allow pacman to use colors, and to show a list of the packages instead of the traditional inline method. Remove the comments under de section `# Misc options`.

<pre>
nano /etc/pacman.conf 
  Color
  VerbosePkgLists
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
cp /my/user/background.jpg /usr/share/sddm/themes/breeze/background.jpg
chmod 644 /usr/share/sddm/themes/breeze/background.jpg
nano /usr/share/sddm/themes/breeze/theme.conf
  [General]
  background=background.jpg
</pre>

#### Bootloader GRUB2 Messages

To display the output of the startup and shutdown boot process, the following flags have to be set in GRUB2. Additionaly, do not forget to remove the default `quiet` flag.

<pre>
nano /etc/default/grub
  GRUB_CMDLINE_LINUX_DEFAULT="noquiet nosplash"

grub-mkconfig -o /boot/grub/grub.cfg 
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
