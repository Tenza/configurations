# Installations

***DO NOT USE THESE NOTES BLINDLY.***  
***SOME CONFIGURANTIONS ARE PERSONAL AND PROBABLY OUTDATED.***

This file is simply my backlog of installations.  
This document does not contain instructions with explanations meant as a guide.

#### KDE5 Desktop Environment

Install `KDE5`, system language and the `SDDM` display manager that better integrates with KDE5.  
Install the official `breeze` theme and packages to give a uniform look to `Qt` and `GTK` applications.  
Install the `VLC` phonon backend, it has the best upstream support compared with GStreamer, although both can be installed.   Install `libx264` instead of `libx264-10bit` because there are some compatibility issues with 10bit encoders.  

If the packages are not installed on a ***single call*** to pacman, some of the packages explicitly set in here will be prompted.

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

For reference, this can be done using a graphical interface inside KDE, because the package `sddm-kcm` was already installed. This setting is in `Setting > Startup and Shutdown > Login Screen. (2nd tab)`.

> If the `sddm --example-config` command refuses to run, try `su - root`.

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/KDE  
https://wiki.archlinux.org/index.php/desktop_environment  
https://wiki.archlinux.org/index.php/window_manager  
http://forum.doom9.org/showthread.php?t=167654
</sup></sub>

#### Activate already installed functionalities

The `plasma-meta` package comes with the following dependencies:

<pre>
bluedevil
breeze-gtk
drkonqi
kde-gtk-config
kdeplasma-addons
kgamma5
kinfocenter
kscreen
ksshaskpass
ksysguard
kwallet-pam
kwayland-integration
kwrited
oxygen
plasma-desktop
plasma-mediacenter
plasma-nm
plasma-pa
plasma-sdk
plasma-workspace-wallpapers
powerdevil
sddm-kcm
user-manager
</pre>

##### Network

The `NetworkManager` package is responsible for managing the network functionalities, and it was already installed as a dependency to the KDE `plasma-nm` front-end. Simply `status/start/enable` the service if needed. Also, if the `dhcpcd` or `netctl-auto` services were enabled, they need to be [stopped and disabled](https://github.com/Tenza/configurations/blob/master/ArchLinux%20Installation/2%20-%20Base%20System%20Configuration.md#optional-not-recommended-start-network-connections-at-boot) first to avoid conflicts.

<pre>
systemctl start NetworkManager.service
</pre>

NetworkManager by default stores passwords in clear text in the connection files at `/etc/NetworkManager/system-connections/`.
It is preferable to save the passwords in encrypted form instead of clear text, this can be achieved by storing them in a keyring which NetworkManager then queries for the passwords. For KDE, the keyring daemon is KDE Wallet.

##### KWallet

The `kwallet` and `kwallet-pam`packages should already be installed as a dependency to the `kio` and `plasma-meta` packages. And once the network connects or any other service that requires a keyring deamon, a configuration window should popup, and a wallet should be created. 

KWallet will prompt for the password every first time a application that integrates with `kwallet` requires it. To unlock the wallet automatically at boot, the chosen password has to be either black or the same as your username password. Also, take into consideation that while GPG encryption is superior, `kwallet-pam` it is not compatible with GnuPG keys, so it is not able to automaticly unlock the wallet at boot. Edit the login manager pam file for SDDM.

<pre>
nano /etc/pam.d/sddm
  auth            include         system-login
  -auth            optional        pam_kwallet5.so
  account         include         system-login
  password        include         system-login
  session         include         system-login
  -session         optional        pam_kwallet5.so
</pre>

##### Bluetooth



#### Install Applications

The `plasma-meta` package comes with the barebones of the KDE desktop enviroment. To install more applications without a graphical console, we can simply switch the TTY by pressing `CTRL+ALT+F1-7`, KDE should be running on the `TTY1`.

<pre>
CTRL+ALT+F2
</pre>

> Note that the default TTY was changed in the migration of `systemd/logind` on October 2012 to [fix a problem](https://bugs.archlinux.org/task/32206). The  is different from KDE4, that used to run on the TTY7


#### KDE Applications

<pre>
pacman -S kconsole dolphin kate
</pre>

