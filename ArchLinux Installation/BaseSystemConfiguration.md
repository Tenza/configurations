# Base System Configuration

***DO NOT USE THESE NOTES BLINDLY.***  
***SOME CONFIGURANTIONS ARE PERSONAL AND PROBABLY OUTDATED.***

#### Configure hostname

This is the name that will show-up in the console, for example: `filipe@filipe-desktop`

<pre>
hostnamectl set-hostname filipe-desktop
</pre>

#### Configure persistent keymap

<pre>
localectl set-keymap pt-latin9
localectl set-x11-keymap pt
localectl status
</pre>

Result:
<pre>
[filipe@filipe-desktop ~]$ localectl status  
System Locale: LANG=pt_PT.UTF-8  
VC Keymap: pt-latin9  
X11 Layout: pt
</pre>

#### Configure timezone 

Get time zone info:  
<pre>
ls /usr/share/zoneinfo
</pre>

In my case is `Portugal`, so:
<pre>
timedatectl set-timezone Portugal
</pre>

#### Configure locale

<pre>
nano /etc/locale.gen
</pre>

Remove the comment from the lines with the country code `pt_PT`, there are 3.

<pre>
locale-gen  
localectl set-locale LANG="pt_PT-UTF-8"
</pre>

#### Start network cards

DHCP is not enabled by default like when we boot from the USB, so we have to start it manually.

Get interfaces name:
<pre>
ip link
</pre>

In my case the interfaces needed are:  
<pre>
Wired: enp2s0  
Wireless: wlp3s0
</pre>

#### Start wired connection with DHCP

<pre>
systemctl start dhcpcd@enp2s0.service
</pre>

#### Start wireless connection with wifi-menu

<pre>
wifi-menu -o wlp3s0
</pre>

###### (Optional) (Not recommended) Start network at boot

This will activate the connections on boot, but keep in mind, that these will have to be **stopped and disabled** when we install the `networkmanager` package once we have a graphical environment.

<pre>
systemctl enable dhcpcd@enp2s0.service  
systemctl enable netctl-auto@wlp3s0.service
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/Beginners%27_guide#Configure_the_network
</sup></sub>

#### (1) Configure hardware clock for UTC

By default linux uses UTC, so we should instal NTP to sync the time online.

<pre>
pacman -S ntp  
systemctl enable ndpd.service
</pre>

##### (2) (Not recommended) Configure hardware clock for Localtime

The only reason to ever do this, is if we are in a dualboot configuration and there is already an OS managing time and DST switching while storing the RTC time in localtime, such as Windows. But even so, it would be preferable to force Windows to use UTC, rather than forcing Linux to localtime. To do this, just install the Meinberg NTP Software.

With this said, keep in mind that we still have to logon in Windows at least two times a year (in Spring and Autumn) when DST kicks i.n

<pre>
timedatectl set-local-rtc true
</pre>

<sub><sup>
References:  
https://wiki.archlinux.org/index.php/Time  
http://www.satsignal.eu/ntp/setup.html  
http://www.meinbergglobal.com/english/sw/ntp.htm
</sup></sub>

#### Dualboot

First we install the package responsible for detecting other OS installations:

<pre>
pacman -S os-prober
</pre>

<sub><sup>
References:
https://wiki.archlinux.org/index.php/GRUB#Dual-booting
</sup></sub>

#### Windows already instaled:

If we are installing Arch and Windows is already on the machine, assuming the bootloader was already installed to the MBR, we simply need to refresh the config file.

<pre>
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

The os-prober will be automatically called by `grub-mkconfig` and will add an entry to Windows. This has worked for me even on a dualboot with a RAID array.

But if for some reason it failed we can try and add a custom entry:

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

If we are installing Windows and Arch is already on the machine, we have to reinstall the bootloader because Windows automatically overwrites the MBR. To do so, we have to start Arch Live and do: 

<pre>
loadkeys pt-latin9
mount /dev/<b>sdX</b> /mnt
arch-chroot /mnt
cfdisk /dev/<b>sdX</b>  (if you need to set the bootflag)
grub-mkconfig -o /boot/grub/grub.cfg
grub-install -target=i386-pc --recheck /dev/<b>sdX</b>
</pre>

If you need any extra package additionally use:
<pre>
wifi-menu
pacman -Syy
pacman -S os-prober
</pre>

#### Custom Entries

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

<pre>
grub-mkconfig -o /boot/grub/grub.cfg
</pre>

We can also change the text displayed on the bootloader if we manually edit the file.
Keep in mind that these changes will be lost when we rebuild the file. **Do this with extreme caution.**
If we delete something needed, the machine might not boot (you can always try to fix it with a live cd).

<pre>
nano /boot/grub/grub.cfg
</pre>

#### Reboot

<pre>
systemctl reboot
</pre>
