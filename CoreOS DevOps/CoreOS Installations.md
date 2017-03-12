## CoreOS Installations

CoreOS is a distribution that targets container enviroments. Unlike other distributions, it is not really expected any interaction with CoreOS itself, other than managing containers. CoreOS comes with minimal packages pre-installed, and a package manager is not available by default. 

After installing CoreOS, all commands will be run through SSH, even locally on the VirtualBox installation. This way it will be possible to simply copy-paste commands, with any terminal, and have the host keyboard layout working as intended. 

### CoreOS Development VirtualBox Installation under Linux

This installation approach is automated by bash scripts provided by CoreOS, and it will be targed to Arch Linux.

#### Installations

Install VirtualBox and it's optional dependencies.

<pre>
pacman -S virtualbox virtualbox-guest-iso virtualbox-host-modules-arch virtualbox-guest-utils
pacman -S vde2 net-tools virtualbox-ext-vnc
</pre>

Install the packages needed to SSH and execute the scripts.

<pre>
pacman -S wget openssh cdrtools
</pre>

Activate the following modules VirtualBox uppon boot.
`vboxnetadp` is needed to create the host interface in the VirtualBox global preferences. 
`vboxnetflt` is needed to launch a virtual machine using that network interface.
`vboxpci` is needed to pass through PCI device on your host.

<pre>
sudo nano /etc/modules-load.d/virtualbox.conf
	vboxdrv
	vboxnetadp
	vboxnetflt
	vboxpci
</pre>

### CoreOS Development VirtualBox Installation under Windows

This installation approach can be reproduced under Windows or Linux. The downside of this method is that it cannot be fully automated using the bash scripts, and a few commands will have to be typed directly on the VitualBox installation, not through SSH.

### CoreOS Production Cloud Installation 

The majority of cloud providers will install CoreOS for you, and you will only have to parse the cloud-config file uppon creation. Simply copy and paste the content of the cloud-config-production.yaml file.
