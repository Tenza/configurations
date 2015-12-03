# Base System XOrg

***DO NOT USE THESE NOTES BLINDLY.***  
***SOME CONFIGURANTIONS ARE PERSONAL AND PROBABLY OUTDATED.***

#### Add new user

It is good practice to use a normal user and elevate to root only when necessary.  
There are a few ways to do it, but the most common ones are the following.

<pre>
useradd -m -G wheel -s /bin/bash filipe 
</pre>

This command will also automatically create a group called `filipe` with the same GID as the UID of the user `filipe`
and makes this the default group for `filipe` on login. Making each user have their own group (with group name same as 
user name and GID same as UID) is the preferred way to add users. 

You could also make the default group something else, e.g. users. 

<pre>
useradd -m -g users -G wheel -s /bin/bash filipe
</pre>

However, using a single default group (users in the example above) is not recommended for multi-user systems. 
The reason is that typically, the method for facilitating shared write access for specific groups of users is 
setting user umask value to 002, which means that the default group will by default always have write access 
to any file you create. 

Change your finger information and password  
<pre>
chfn filipe  
passwd filipe
</pre>

#### Give sudo permissons to new user

Trick visudo to open with nano:
<pre>
EDITOR=nano visudo
</pre>

Add user to the section “User privilege specification”:
<pre>
Filipe ALL=(ALL) ALL
</pre>

Logout from root and enter the new account:
<pre>
logout
</pre>

Once you login with the new user, you can checkout the groups information with the command:
<pre>
id
</pre>

#### Activate multilib repo

Remove comments from [multilib]:
<pre>
sudo nano /etc/pacman.conf  
sudo pacman -Syy
</pre>

##### Install Packer

First install the components needed to use `wget` and `git`, then get the tarball and extract.  
Finally, build the package with makepkg and install it using pacman.

<pre>
sudo pacman -S wget git jshon expac  
wget https://aur.archlinux.org/packages/pa/packer/packer.tar.gz  
tar zxvf packer.tar.gz  
cd packer && makepkg  
sudo pacman -U packer (press tab)
</pre>

Once installed, cleanup:

<pre>
rm -R packer  
rm packer.tar.gz
</pre>

<sub><sup>
References:
http://www.cyberciti.biz/faq/unpack-tgz-linux-command-line/
</sup></sub>

##### Sound drivers

ALSA is already apart of the Kernel, but the channels are muted by default.

<pre>
sudo pacman -S alsa-utils  
run `alsamixer`  
Press H to unmute. Press F1 for help.  
run `speaker-test`
</pre>

(TO CONTINUE)

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
