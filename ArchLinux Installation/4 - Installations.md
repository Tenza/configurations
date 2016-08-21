# Installations

***DO NOT USE THESE NOTES BLINDLY.***  
***SOME CONFIGURANTIONS ARE PERSONAL AND PROBABLY OUTDATED.***

This document does not contain instructions meant as a guide.
This file is simply my backlog of installations.

#### KDE5 Desktop Environment

Install `KDE5`, system language and the `SDDM` display manager.  
Install the official `breeze` theme and packages to give a uniform look to `Qt` and `GTK` applications.  
Install the `VLC` Phonon multimedia API, it has the best upstream support compared with GStreamer, although both can be installed. Also install `libx264` instead of `libx264-10bit` because there are some compatibility issues with 10bit encoders.  
Finally, enable the `SDDM.service` to start at boot.

If the packages are not installed on a single call to pacman, some of the packages explicitly set in here will be prompted.

<pre>
pacman -S plasma-meta kde-l10n-pt sddm sddm-kcm
pacman -S breeze breeze-kde4 breeze-gtk kde-gtk-config 
pacman -S phonon-qt5 phonon-qt5-vlc libx264
systemctl enable sddm.service
</pre>

To better integrate SDDM with Plasma, it is recommended to edit `/etc/sddm.conf` to use the `breeze` theme.  
Edit this file manually before the first boot, there are some difficulties starting KDE with the SDDM `maui` default theme.  
This can be done using a graphical interface inside KDE, because the package `sddm-kcm` was already installed. This setting is in `Setting > Startup and Shutdown > Login Screen. (2nd tab)`.

<pre>
sddm --example-config > /etc/sddm.conf
sudo nano /etc/sddm.conf
  [Theme]
  Current=breeze
  CursorTheme=breeze_cursors
  FacesDir=/usr/share/sddm/faces
  ThemeDir=/usr/share/sddm/themes
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/KDE  
https://wiki.archlinux.org/index.php/desktop_environment  
https://wiki.archlinux.org/index.php/window_manager  
http://forum.doom9.org/showthread.php?t=167654
</sup></sub>
