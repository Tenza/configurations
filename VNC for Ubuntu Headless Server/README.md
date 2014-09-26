## Add VNC Access

Installation of X11VNC with the graphical environment LXDE.

The configuration files are prepared to be used with a VNC connection through SSH gateway.  
Just run the command an then use localhost:5900 through ip_address:22  

### Using files from GitHub

Replace the **PASSWORD** on the last line with the password for the VNC access. 

>sudo apt-get update &&  
sudo apt-get -y dist-upgrade &&  
sudo apt-get -y install curl xorg lxde x11vnc xserver-xorg-video-dummy &&  
curl -o /etc/init.d/x11vncserver https://raw.githubusercontent.com/Tenza/configurations/master/VNC%20for%20Ubuntu%20Headless%20Server/x11vncserver &&   
curl -o /etc/X11/xorg.conf https://raw.githubusercontent.com/Tenza/configurations/master/VNC%20for%20Ubuntu%20Headless%20Server/xorg.conf &&  
chmod +xr /etc/init.d/x11vncserver &&  
update-rc.d x11vncserver defaults &&  
mkdir /root/.vnc &&  
x11vnc -storepasswd **PASSWORD** /root/.vnc/passwd &&  
reboot

### Using files from Dropbox

This command assumes that the **Configuration wizard** of the Dropbox-Uploader script has already been used, and that the auth file **.dropbox\_uploader** has already been generated, [more information here.](https://github.com/andreafabrizi/Dropbox-Uploader/)     
With this in mind, just add the file **.dropbox\_uploader** to **/root/** and make sure that the configuration files are in place.  

Replace the **PASSWORD** on the last line with the password for the VNC access.   

>sudo apt-get update &&  
sudo apt-get -y dist-upgrade &&  
sudo apt-get -y install git xorg lxde x11vnc xserver-xorg-video-dummy &&  
cd /root/ &&  
git clone https://github.com/andreafabrizi/Dropbox-Uploader/ &&  
cd /root/Dropbox-Uploader &&  
./dropbox\_uploader.sh download x11vncserver /etc/init.d &&  
./dropbox\_uploader.sh download xorg.conf /etc/X11 &&  
chmod +xr /etc/init.d/x11vncserver &&  
update-rc.d x11vncserver defaults &&  
mkdir /root/.vnc &&  
x11vnc -storepasswd **PASSWORD** /root/.vnc/passwd &&  
reboot

## Remove VNC Access
> update-rc.d -f x11vncserver remove &&  
rm /etc/init.d/x11vncserver &&  
rm -d -r /root/.vnc/ &&  
reboot  

## x11vncserver Commands
> service x11vncserver start  
service x11vncserver stop

##Personal files

### Getting personal files from dropbox

The above requirements apply.

> mkdir /root/MyProgram &&  
cd /root/Dropbox-Uploader &&  
./dropbox\_uploader.sh download MyProgram /root/MyProgram &&  
chmod +x /root/MyProgram/MyProgram

### Updating personal files from dropbox

The above requirements apply.

> rm /root/MyProgram/MyProgram &&  
cd /root/Dropbox-Uploader &&  
./dropbox\_uploader.sh download MyProgram /root/MyProgram &&  
chmod +x /root/MyProgram/MyProgram 
