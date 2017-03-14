# CoreOS Installations

CoreOS is a distribution that targets container environments. Unlike other distributions, it is not really expected any interaction with CoreOS itself, other than managing containers. CoreOS comes with minimal packages pre-installed, and a package manager is not available by default. 

After installing CoreOS, all commands will be run through SSH, even locally on the VirtualBox installation. This way it will be possible to simply copy-paste commands, with any SSH client, and have the host keyboard layout working as intended. 

# CoreOS VirtualBox Installation under Linux

This installation approach is automated by bash scripts provided by CoreOS, and it will be targeted to ArchLinux. This approach also has the advantage of using a ConfigDrive, that makes it easier to update the `cloud-config` file in use.

#### VirtualBox Installation

Install VirtualBox and it's optional dependencies.

<pre>
pacman -S virtualbox virtualbox-guest-iso virtualbox-host-modules-arch virtualbox-guest-utils
pacman -S vde2 net-tools virtualbox-ext-vnc
</pre>

Install the packages needed to SSH and execute the scripts.

<pre>
pacman -S wget openssh cdrtools
</pre>

Activate the following VirtualBox modules upon boot.  
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

#### Create VirtualBox VDI File and ConfigDrive

The following chain of commands will download and execute the scripts copied from CoreOS repo. The result should be a `CoreOS-VDI` folder with the files `coreos_production_stable.vdi` and `coreos_configdrive.iso`.

Before execution, make sure your public key is at `~/.ssh/id_rsa.pub`. To generate one, simply run `ssh-keygen`.

<pre>
mkdir CoreOS-VDI &&
cd CoreOS-VDI/ &&
curl -O https://raw.githubusercontent.com/Tenza/configurations/master/CoreOS%20DevOps/create-coreos-vdi &&
chmod +x create-coreos-vdi &&
./create-coreos-vdi -V stable &&
find . -name '*.vdi' -exec mv {} coreos_production_stable.vdi \; &&
curl -O https://raw.githubusercontent.com/Tenza/configurations/master/CoreOS%20DevOps/create-basic-configdrive &&
chmod +x create-basic-configdrive &&
./create-basic-configdrive -H coreos_configdrive -S <b>~/.ssh/id_rsa.pub</b> &&
rm create-basic-configdrive &&
rm create-coreos-vdi
</pre>

<sub><sup>
Original Links:  
https://raw.githubusercontent.com/coreos/scripts/master/contrib/create-coreos-vdi  
https://raw.githubusercontent.com/coreos/scripts/master/contrib/create-basic-configdrive  
</sup></sub>

#### VirtualBox Setup

<pre>
<b>Create a new VM in VirtualBox.</b>
    Choose Linux 2.6 / 3.x / 4.x (64 bit).
    Give at least 1024MB of memory.
    Select the generated VDI file.
<b>Go to the VM settings.</b>
    Go to the Network tab.
    Make the VM visible on the local network by choosing <b><a href="https://www.howtogeek.com/122641/how-to-forward-ports-to-a-virtual-machine-and-use-it-as-a-server/">Bridged Mode</a></b>.
    Go to the Storage tab.
    Add a new optical drive and select the generated .iso file.
</pre>

#### SSH Into that Box

Power on the VM, and SSH into the IP displayed at the top of the VM screen.

<pre>
ssh core@192.168.1.2
</pre>

# CoreOS VirtualBox Installation under Windows

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

It will now be necessary to add a `cloud-config.yaml` file to the machine. This can be either manually typed out or downloaded. The following command will download a minimal `cloud-config.yaml` from github, with the user `core` and password `core`, simply to be able to logon the machine after the installation. 

<pre>
curl -O https://raw.githubusercontent.com/Tenza/configurations/master/CoreOS%20DevOps/cloud-config.yaml
</pre>

###### (Optional) SSH Key

The MD5 `core` password in the file above was generated using `openssl passwd -1`, and it can be replaced at will.

It is also possible to use an SSH Key. To do so, start by generating a new key-pair with `ssh-keygen`, and then simply add the public-key to your `cloud-config.yaml`. Don't forget to [validate the file](https://coreos.com/validate/), before transferring it to the VM.

<pre>
#cloud-config
ssh_authorized_keys:
  - ssh_rsa AAAABBBBCCCC...
</pre>

#### Install CoreOS

Before executing the install command, make sure the drive is actually in `/dev/sda` by executing the `fdisk -l`. The `-C` flag specifies the channel (it actually uses stable by default) and the `-c` flag with the path of the config file. 

<pre>
coreos-install -d /dev/sda -C stable -c /home/core/cloud-config.yaml
</pre>

Once the following message is displayed, the machine should be powered off, and the ISO unmounted.

<pre>
<b>Success! CoreOS stable $version is installed on /dev/sda.</b>
poweroff

Go to the VM settings.
Storage tab.
Remove the optical drive with the ISO image.
</pre>

#### SSH Into that Box

Power on the VM, and login with the username `core` and password `core` to enable to SSH daemon.  

<pre>
systemctl start/status/enable sshd
ifconfig
</pre>

The SSH server should be up and running.

<pre>
ssh core@192.168.1.2
</pre>
