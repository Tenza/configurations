# Base System Configuration

***DO NOT USE THESE NOTES BLINDLY.***  
***SOME CONFIGURANTIONS ARE PERSONAL AND PROBABLY OUTDATED.***

This is a follow-up of the [Base System Installation](https://github.com/Tenza/configurations/blob/master/ArchLinux%20Installation/1%20-%20Base%20System%20Installation.md) guide.  
The configurations set in here suppose that ArchLinux is properly installed on the system.

#### Configure hostname

Let's start by setting the hostname, this is the name that will show-up in the console, for example: `filipe@filipe-desktop`.
The switch `set-hostname` actually sets the three hostnames that `hostnamectl` manages.

<pre>
hostnamectl set-hostname filipe-desktop
</pre>

<sub><sup>
References: 
https://www.freedesktop.org/software/systemd/man/hostnamectl.html
</sup></sub>

#### Configure locale

To configure the locale, start by editing the file `locale.gen` and remove the comment from the lines with the desired country code. In my case, it's `pt_PT`, and there are 3 commented lines.

<pre>
nano /etc/locale.gen
</pre>

It is now possible to generate and set that locales, note that previously the command `loadkeys` was used to load the keyboard layout, but that configuration was lost when the live system exited. To make it persistent `localectl` has to be used.  

Also, keep in mind that what `set-locale` actually does, is add the text passed as an argument, to the file `/etc/locale.conf`, the text has to be one of the uncommented lines from the previous step, in this case `pt_PT-UTF-8` will be used.  
If in doubt, use the list functions available within `localectl`.

<pre>
locale-gen  
localectl set-locale LANG="pt_PT-UTF-8"
localectl set-keymap pt-latin9
localectl set-x11-keymap pt
localectl status 
</pre>

If the locale is properly configured, this should be the output for the command `localectl status`.

<pre>
[filipe@filipe-desktop ~]$ localectl status  
System Locale: LANG=pt_PT.UTF-8  
VC Keymap: pt-latin9  
X11 Layout: pt
</pre>

> The `mandb` warning from the previous guide should be solved by now.

<sub><sup>
References: 
https://www.freedesktop.org/software/systemd/man/localectl.html
</sup></sub>

#### Configure timezone 

The timezone will be used to determine the localtime correctly.  
What `timedatectl set-timezone` actually does is create a `/etc/localtime` symlink that points to a zoneinfo file under `/usr/share/zoneinfo/` and if the Real Time Clock (RTC) is configured to be in the localtime, this will also update the RTC time.  

<pre>
timedatectl list-timezones
timedatectl set-timezone Europe/Lisbon
</pre>

<sub><sup>
References: 
https://www.freedesktop.org/software/systemd/man/timedatectl.html
</sup></sub>

#### (1) Configure hardware clock for UTC

By default linux uses UTC, so a NTP daemon is required to sync the time online.
The `systemd-timesyncd` service is available with systemd, use `timedatectl` to start and enable it.

<pre>
timedatectl set-ntp true 
</pre>

There are other implementations of the NTP protocol, but for a simple client side, focusing only on querying time from one remote server, `systemd-timesyncd` should be more than appropriate for most installations. For server related functionalities, check out the [ntp](https://wiki.archlinux.org/index.php/Network_Time_Protocol_daemon) package.

#### (2) (Not recommended) Configure hardware clock for Localtime

The only reason to ever do this, is if the system is in a dualboot configuration and there is already an OS managing time and DST switching while storing the RTC time in localtime, such as Windows. 

But even so, it would be preferable to force Windows to use UTC, rather than forcing Linux to localtime. There are registry modifications as well as software to manage time in UTC under Windows, consider for instance installing the [Meinberg NTP Software](http://www.meinbergglobal.com/english/sw/ntp.htm). Also, keep in mind that the OS managing time still needs to be booted to update the RTC clock at least two times a year (in Spring and Autumn) when DST kicks in.

To still force Linux use to localtime, set `set-local-rtc` to `true`.

<pre>
timedatectl set-local-rtc true
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Time  
https://www.freedesktop.org/software/systemd/man/timedatectl.html  
http://www.satsignal.eu/ntp/setup.html  
http://www.meinbergglobal.com/english/sw/ntp.htm
</sup></sub>

#### Network interfaces

##### Wired interface

DHCP is not enabled by default like when Archlive was booted, it has to be started manually.  
First identify the name of the network interface and then start the DHCP service.
In my case, the wired interface is `enp2s0`.

<pre>
ip link
systemctl start dhcpcd@enp2s0.service
</pre>

##### Wireless interface

The wireless interface, can be started exactly like Archlive, if the [needed packges were installed](https://github.com/Tenza/configurations/blob/master/ArchLinux%20Installation/1%20-%20Base%20System%20Installation.md#optional-install-necessary-wireless-drivers).  
Keep in mind that `wifi-menu` is a tool included with `netctl`, and once it is used to connect to a network, a profile will be saved under `/etc/netctl` with the network details. 

<pre>
wifi-menu
</pre>

This profile can be used by `netctl-auto` to auto connect to a network. If a profile is already available, it is possible to simply start `netctl-auto` to use any of the saved profiles. In my case, the wireless interface is `wlp3s0`.

<pre>
ip link
systemctl start netctl-auto@wlp3s0.service
</pre>

#### (Optional) (Not recommended) Start network connections at boot

The above commands can be enabled at boot to activate the connections. But note that these will have to be **stopped and disabled** when another package is installed to manage the network (for instance, the package `networkmanager` installed on the KDE graphical environment), otherwise it will likely cause a disruption in connectivity. 

<pre>
systemctl enable dhcpcd@enp2s0.service  
systemctl enable netctl-auto@wlp3s0.service
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Beginners%27_guide#Configure_the_network
</sup></sub>

#### Pacman

The default mirrorlist of pacman is sorted by the synchronization status and speed at the time the installation image was created. The quality of mirrors can vary over time so it is a good idea to have a package to deal with this. First install the `reflector` package, and than run `reflector` to rate the 25 most recently synchronized HTTPS servers, sort them by download rate, and overwrite the file `/etc/pacman.d/mirrorlist`.

<pre>
pacman -S reflector
reflector --verbose -l 25 -p https --sort rate --save /etc/pacman.d/mirrorlist
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Mirrors  
https://wiki.archlinux.org/index.php/Reflector
</sup></sub>

##### Activate [multilib] repository

The multilib repository is an official repository which allows the user to run and build 32-bit applications on 64-bit installations of Arch Linux. A 64-bit installation of Arch Linux with multilib enabled follows a directory structure similar to Debian. The 32-bit compatible libraries are located under `/usr/lib32/`, and the native 64-bit libraries under `/usr/lib/`.

To use the multilib repository, uncomment the `[multilib]` section in `/etc/pacman.conf`, be sure to uncomment both lines.

<pre>
nano /etc/pacman.conf  
pacman -Syy
</pre>

## Dualboot

The package responsible for detecting other OS installations is `os-prober`, and it should be installed before any additional configurations.

<pre>
pacman -S os-prober
</pre>

> I had a problem where `os-prober` tried to mount my Linux extended partition, probably because it is a mapped partition given that this is a RAID system. The detection of other OS worked, and the entry was added, but a error showed up when it tried to examine that particular partition.

<sub><sup>
References:
https://wiki.archlinux.org/index.php/GRUB#Dual-booting
</sup></sub>

#### Windows already installed

If Arch was installed while Windows was already on the machine, and assuming the GRUB2 bootloader was properly installed to the MBR, the only thing left to do, is refresh the GRUB2 configuration file.

<pre>
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

The `os-prober` will be automatically called by `grub-mkconfig` and it will add an entry to Windows on the bootloader menu.

This has worked for me even on a dualboot with a RAID array over dm-crypt, but if for some reason `os-prober` fails to detect other OS, it is possible to manually add a custom entry in `40_custom` file. Start by getting the uuid of the disk, and replace the `XXXXXXXXXXXXXX` on the menuentry, lastly refresh the GRUB2 configuration file.

<pre>
blkid
nano /etc/grub.d/40_custom
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

<pre>
menuentry{
	echo “Loading Windows 7”
	insmod part_msdos
	insmod ntfs
	search --fs-uuid --no-floppy -set=root XXXXXXXXXXXXXX
	chainloader +1
}
</pre>

#### Arch already installed

If Windows was installed while Arch was already on the machine, the bootloader has to be reinstalled because Windows automatically overwrites the MBR. To do so, start Arch Live to enter `chroot` and reinstall the bootloader to the first sector of the disk.

<pre>
loadkeys pt-latin9
mount /dev/<b>sdX</b> /mnt
arch-chroot /mnt
cfdisk /dev/<b>sdX</b>  (if needed to set the bootflag)
grub-mkconfig -o /boot/grub/grub.cfg
grub-install -target=i386-pc --recheck /dev/<b>sdX</b>
</pre>

#### (Optional) Custom Entries

The following custom entries might be worth adding to the GRUB2 menu.

<pre>
nano /etc/grub.d/40_custom
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

<pre>
menuentry "System restart" {
	reboot
}

menuentry "System shutdown" {
	echo "System shutting down..."
	halt
}
</pre>

> The `halt` command might not work if Plug and Play (PnP) is enabled in the BIOS/UEFI settings, in that case the power button has to be pressed. Also, even without PnP the shutdown might take a few seconds, that's why an `echo` message is displayed.

#### (Optional) Change the displayed text in the GRUB menu. 

The displayed text set by `grub-mkconfig` and `os-prober` can be changed by manually editing the `/boot/grub/grub.cfg` file.
Just keep in mind that these changes will be lost when the file is rebuilded by `grub-mkconfig`. Also, edit **with caution** because if something needed is deleted, the system might not boot, and the file will have to be manually fixed with a live cd.

#### Reboot

<pre>
systemctl reboot
</pre>
