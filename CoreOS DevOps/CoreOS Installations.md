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

Activate the following VirtualBox modules uppon boot.
`vboxdrv` is the only mandatory virtualbox module, which must be loaded before any virtual machines can run.
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

#### VirtualBox Setup

<pre>
<b>Download and install VirtualBox.</b>
<b>Create a new VM in VirtualBox.</b>
    Choose Linux 2.6 / 3.x / 4.x (64 bit).
    Give at least 1024MB of memory.
    Create a virtual hard disk.
    Choose the native <a href="https://superuser.com/questions/360517/what-disk-image-should-i-use-with-virtualbox-vdi-vmdk-vhd-or-hdd">VDI</a> format.
<b>Go to the VM settings.</b>
    Go to the Network tab.
    Make the VM visible on the local network by choosing <b><a href="https://www.howtogeek.com/122641/how-to-forward-ports-to-a-virtual-machine-and-use-it-as-a-server/">Bridged Mode</a></b>.
<b>Start the VM and select the downloaded ISO.</b>
    If you are not prompted to select the ISO.
    	Go to Settings.
		Storage tab.
		Add a new optical drive with the ISO image.
</pre>

#### Cloud-config file

It will now be necessary to add a `cloud-config.yaml` file to the machine. This can be either manually typed out or downloaded. The following command will download a minimal `cloud-config.yaml` from github, with the user `core` and password `core`, simply to be able to logon the machine after the instalation. 

<pre>
curl -O link_to_git_page
</pre>

(Optional) The MD5 password in the file was generated with the `openssl passwd -1` command. An SSH Key could also be used, to do so, generate a new pair with `ssh-keygen`. Then build your `cloud-config.yaml` with the public key, like below, then upload the file somewhere, and finally download it inside the VM.

<pre>
#cloud-config
ssh_authorized_keys:
  - ssh_rsa AAAABBBBCCCC...
</pre>

#### Install CoreOS

Before executing the install command, make sure the drive is actually in `/dev/sda` by executing the `fdisk -l`.

<pre>
coreos-install -d /dev/sda -C stable -c ~/cloud-config.yaml
</pre>

### CoreOS Production Cloud Installation 

The majority of cloud providers will install CoreOS for you, and you will only have to parse the cloud-config file uppon creation. Simply copy and paste the content of the cloud-config-production.yaml file.
