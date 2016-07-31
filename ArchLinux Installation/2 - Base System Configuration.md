# Base System Configuration

***DO NOT USE THESE NOTES BLINDLY.***  
***SOME CONFIGURANTIONS ARE PERSONAL AND PROBABLY OUTDATED.***

This is a followup of the [Base System Installation](https://github.com/Tenza/configurations/blob/master/ArchLinux%20Installation/1%20-%20Base%20System%20Installation.md) guide.  
The configurations set in here supose that ArchLinux is properly installed on the system.

#### Configure hostname

This is the name that will show-up in the console, for example: `filipe@filipe-desktop`

<pre>
hostnamectl set-hostname filipe-desktop
</pre>

#### Configure locale

To configure the locale, start by editing the file `locale.gen` and remove the comment from the lines with the desired country code. In my case, it's `pt_PT`, and there are 3 of them. 

<pre>
nano /etc/locale.gen
</pre>

Now generate and set that locales.
Note that previously the command `loadkeys` was used to load the keyboard layout, but that configuration was lost when the live system exited. To make it persistent `localectl` has to be used.  
Also keep in mind that what `set-locale` actualy does, is add the text passed to the file `/etc/locale.conf`, the text has to be one of the uncommented lines from the previous step.  
List functions are also available within `localectl`.

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

#### Configure timezone 

Set the timezone as it will be used to determine the localtime correctly.  
This will create an `/etc/localtime` symlink that points to a zoneinfo file under `/usr/share/zoneinfo/`.  

<pre>
timedatectl list-timezones
timedatectl set-timezone Europe/Lisbon
</pre>

#### Start network cards

##### Wiered interface

DHCP is not enabled by default like Archlive, so it has to be started manually.  
Identify the name of the network interface and then start the DHCP service.
In my case, the wired interface is `enp2s0`. wlp3s0

<pre>
ip link
systemctl start dhcpcd@enp2s0.service
</pre>

##### Wireless interface

The wireless interface, can be started exactly like Archlive.
The `-o` flag can be additionaly used, to save the profile in `netctl`.

<pre>
wifi-menu -o wlp3s0
</pre>

#### (Optional) (Not recommended) Start network at boot

This will activate the connections at boot, but keep in mind, that these will have to be **stopped and disabled** when the package  `networkmanager` is instaled with the KDE graphical environment.

<pre>
systemctl enable dhcpcd@enp2s0.service  
systemctl enable netctl-auto@wlp3s0.service
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Beginners%27_guide#Configure_the_network
</sup></sub>

#### (1) Configure hardware clock for UTC

By default linux uses UTC, so a NTP deamon is required to sync the time online.

<pre>
pacman -S ntp  
systemctl enable ndpd.service
</pre>

#### (2) (Not recommended) Configure hardware clock for Localtime

The only reason to ever do this, is if the system is in a dualboot configuration and there is already an OS managing time and DST switching while storing the RTC time in localtime, such as Windows. But even so, it would be preferable to force Windows to use UTC, rather than forcing Linux to localtime. To do this, just install the Meinberg NTP Software.

If you still want to force Linux use localtime, keep in mind that the OS managing time needs to be initialized to update the RTC clock at least two times a year (in Spring and Autumn) when DST kicks in.

<pre>
timedatectl set-local-rtc true
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Time  
http://www.satsignal.eu/ntp/setup.html  
http://www.meinbergglobal.com/english/sw/ntp.htm
</sup></sub>

## Dualboot

The package responsible for detecting other OS installations is `os-prober`, and it should be installed before any aditional configuration.

<pre>
pacman -S os-prober
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/GRUB#Dual-booting
</sup></sub>

#### Windows already instaled:

If Arch was installed while Windows was already on the machine, and assuming the GRUB2 bootloader was already installed to the MBR, we simply need to refresh the config file.

<pre>
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

The os-prober will be automatically called by `grub-mkconfig` and it will add an entry to Windows on the bootloader menu. This has worked for me even on a dualboot with a RAID array over dm-crypt. but I also report a problem where `os-prober` tries to mount my Linux extended partition, although everything was properly configured, a error still showed up.

But if for some reason `os-prober` fails to detect other OS, it is possible to add a custom entry in `40_custom` file.

<pre>
nano /etc/grub.d/40_custom
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

Use `blkid` to get the uuid of the disk and replace the 'XXX'. And re-run the command:

<pre>
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

#### Arch already instaled:

If Windows was installed while Arch was already on the machine, the bootloader has to be reinstalled because Windows automatically overwrites the MBR. To do so, start Arch Live to enter `chroot` and reinstall the bootloader to the first sector of the disk.

<pre>
loadkeys pt-latin9
mount /dev/<b>sdX</b> /mnt
arch-chroot /mnt
cfdisk /dev/<b>sdX</b>  (if needed to set the bootflag)
grub-mkconfig -o /boot/grub/grub.cfg
grub-install -target=i386-pc --recheck /dev/<b>sdX</b>
</pre>

#### Custom Entries

Handy custom entries might be a good optino

> nano /etc/grub.d/40_custom

<pre>
menuentry "System restart" {
	echo "System rebooting..."
	reboot
}

menuentry "System shutdown" {
	echo "System shutting down..."
	halt
}
</pre>

WARNING: The `halt` command might not work if Plug and Play (PnP) is enabled in the BIOS/UEFI settings, in that case the power button has to be pressed.

<pre>
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

We can also change the text displayed on the bootloader if we manually edit the file.
Keep in mind that these changes will be lost when we rebuild the file. **Do this with extreme caution.**
If something needed is deleted, the machine might not boot. Although it can probably be fixed with live cd.

<pre>
nano /boot/grub/grub.cfg
</pre>

#### Reboot

<pre>
systemctl reboot
</pre>
