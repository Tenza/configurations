# Vodafone VLans Configuration

#### TL;DR  
1. Flash Merlin on the router.
2. Configurate:
   * LAN → IPTV → Manual → Internet: VID 100 | PRIO 0 
   * Administration → System → Enable JFFS custom scripts and configs: Yes 
   * Administration → System → Enable SSH: Yes
3. Reboot.
4. SSH into the router:
   * `vi /jffs/scripts/services-start`
   * Type `i` to enter edit mode, then paste the code bellow with your SSH client paste shortcut.  
Press `esc` to exit edit mode, type `:` to enter command mode, and type `wq` to save and exit.

```bash
#!/bin/sh
logger "Starting configuration of VLans 100 (NET) 101 (VOIP) and 105 (IPTV)"

logger "Creating VLans 101 and 105"
vconfig add eth0 101
vconfig add eth0 105

logger "Enabling VLans 101 and 105"
ifconfig vlan101 up
ifconfig vlan105 up

logger "Creating trunk port 4 for VLans 100, 101 and 105"
robocfg vlan 1 ports "1 2 3 8t"
robocfg vlan 100 ports "0t 4t 8t"
robocfg vlan 101 ports "0t 4t 8t"
robocfg vlan 105 ports "0t 4t 8t"

logger "Done. Secondary router should be functional."
```

#### Long Version 

I recently changed my ISP to Vodafone, replaced the coaxial wiring with fiber optic and during the installation 
I explicitly asked them to **change the new Huawei HG8247H**, that is ONT+Router an on a single device, 
for a separate ONT and Router because I wanted to have my **Asus RT-AC66U as the primary router**. 

The hardware installed was:

| Device        | Name                    | 
| ------------- |:-----------------------:| 
| ONT           | Huawei EchoLife HG8012H | 
| Router        | Technicolor TG784n V3   | 
| STB           | Cisco ISB2231           |
| Phone         | Philips M330            | 

**Drawbacks** on having two separate devices:  
1. More power consumption, two devices instead of one.  
2. The Technicolor has only one gigabit port, not four like the new Huawei. But in my case, 
it makes no difference because this router will only have the STB physically connected to it.

**Advantages** of having two separate devices:  
1. You have the ability to choose your primary router.  
2. Much more freedom and functionality with your own router as primary.  
3. Extended wireless signal, my routers are far apart.  
4. Load balanced between routers.  

I want my setup to be the following:

[![Click here to view the image!](http://s30.postimg.org/d9nlgzzpd/Network_Diagram.jpg)](http://s30.postimg.org/d9nlgzzpd/Network_Diagram.jpg)

In order to accomplish this, we have to know the settings of the Technicolor router, go to:
* Web-interface at **192.168.1.254**
* Click on the interfaces
* Click on show details top left corner

| VLAN        | Functionality | 
| ------------|:-------------:| 
| VLAN 100    | Web           | 
| VLAN 101    | VOIP          | 
| VLAN 105    | IPTV          |

Is also worth mentioning that the Technicolor as **Auto MDI-X ports**, 
the interfaces detects if the connection would require a crossover cable, 
and automatically chooses the MDI or MDI-X configuration to properly match the other end of the link. 
**In other words, we don't need a crossover cable to connect the routers, a straight through cable will work just fine.**

Now, we will need to set up a **trunk port to merge the VLANs to a single physical port**. 
In order to create this trunk, we will have to change the firmware of the router, 
because the stock firmware has very limited functionalities regarding VLANs and Trunks.  

We have a few options here, **the most popular ones are DD-WRT and Tomato** and both allow us 
to do this with their web-interface. But I wanted to maintain the functionalities of the original firmware, 
so I choose **Merlin**, that is an enhanced version of the AsusWrt firmware.

#### Flash Merlin to Asus RT-AC66U 

I started by flashing the Merlin firmware on my router, this is very easy, is just like flashing the original firmware. 
https://github.com/RMerl/asuswrt-merlin/wiki/Installation

#### Properly configure Merlin 

Then, the router needs to be configured to get it's Internet access via VLAN100, 
rather than the default untagged network. This can be done in the web-interface: 
* Advanced Settings → LAN → IPTV 
* Select ISP Profile: Manual 
* Internet: VID 100 | PRIO 0

You should now have internet access on your Asus. 

We are going to need SSH access to the router, because the settings that we are about to apply cannot be done through 
the web-interface, just like the original firmware so:
* Advanced Settings → Administration → System 
* Enable JFFS custom scripts and configs: Yes
* Enable SSH: Yes
* Reboot the router

#### SSH into that box

Now that we have SSH, use your favorite SSH client to access the router, I will be using Remmina.  
The server is **192.168.1.1 or router.asus.com**, and login with the same web-interface user.

You can now type the following commands manually in order to test them, 
but remember that **they will not survive at reboot**, therefor we are going to need a script so that they execute in every boot. 

Current **robocfg** configurations (MACs replaced):
```
filipe@RT-AC66U:/tmp/home/root# robocfg show
Switch: enabled gigabit
Port 0: 1000FD enabled stp: none vlan: 2 jumbo: off mac: 00:00:00:00:00:00
Port 1: 1000FD enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
Port 2:   DOWN enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
Port 3:   DOWN enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
Port 4:   DOWN enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
Port 8: 1000FD enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
VLANs: BCM53115 enabled mac_check mac_hash
   1: vlan1: 1 2 3 4 8t
   2: vlan2: 0 8u
 100: vlan100: 0t 8t
```

Current **ifconfig** configurations (IPs and MACs replaced):
```
filipe@RT-AC66U:/tmp/home/root# ifconfig
br0        Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           inet addr:192.168.1.1  Bcast:192.168.1.255  Mask:255.255.255.0
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:5339 errors:0 dropped:0 overruns:0 frame:0
           TX packets:6031 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:811360 (792.3 KiB)  TX bytes:2459930 (2.3 MiB)

eth0       Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:9584 errors:0 dropped:0 overruns:0 frame:0
           TX packets:10711 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000 
           RX bytes:2998645 (2.8 MiB)  TX bytes:4400203 (4.1 MiB)
           Interrupt:4 Base address:0x2000 

eth1       Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:524 errors:0 dropped:0 overruns:0 frame:12427
           TX packets:1775 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000 
           RX bytes:84365 (82.3 KiB)  TX bytes:362567 (354.0 KiB)
           Interrupt:3 Base address:0x8000 

eth2       Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:0 errors:0 dropped:0 overruns:0 frame:0
           TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000 
           RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
           Interrupt:5 Base address:0x8000 

lo         Link encap:Local Loopback  
           inet addr:127.0.0.1  Mask:255.0.0.0
           UP LOOPBACK RUNNING MULTICAST  MTU:16436  Metric:1
           RX packets:44 errors:0 dropped:0 overruns:0 frame:0
           TX packets:44 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:6948 (6.7 KiB)  TX bytes:6948 (6.7 KiB)

vlan1      Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:6396 errors:0 dropped:0 overruns:0 frame:0
           TX packets:7846 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:1073648 (1.0 MiB)  TX bytes:3940581 (3.7 MiB)

vlan100    Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           inet addr:YOUR.PUBLIC.IP  Bcast:YOUR.BCAST.IP  Mask:255.255.128.0
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:1365 errors:0 dropped:0 overruns:0 frame:0
           TX packets:1269 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:260848 (254.7 KiB)  TX bytes:135910 (132.7 KiB)

wl0.1      Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:0 errors:0 dropped:0 overruns:0 frame:12427
           TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000 
           RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

As you can see, there is no VLAN 101 or 105, that we need for VOIP and IPTV, so lets create them:
```
filipe@RT-AC66U:/tmp/home/root# vconfig add eth0 101
filipe@RT-AC66U:/tmp/home/root# vconfig add eth0 105

filipe@RT-AC66U:/tmp/home/root# ifconfig vlan101 up
filipe@RT-AC66U:/tmp/home/root# ifconfig vlan105 up
```

Here we are simply creating the VLans 101 and 105 using [vconfig](http://linux.die.net/man/8/vconfig) command
and enabling the interfaces with [ifconfig](http://linux.die.net/man/8/ifconfig).

If we **ifconfig** again (IPs and MACs replaced):
```
filipe@RT-AC66U:/tmp/home/root# ifconfig
br0        Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           inet addr:192.168.1.1  Bcast:192.168.1.255  Mask:255.255.255.0
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:5533 errors:0 dropped:0 overruns:0 frame:0
           TX packets:6227 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:841635 (821.9 KiB)  TX bytes:2534189 (2.4 MiB)

eth0       Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:9815 errors:0 dropped:0 overruns:0 frame:0
           TX packets:10969 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000 
           RX bytes:3045107 (2.9 MiB)  TX bytes:4484440 (4.2 MiB)
           Interrupt:4 Base address:0x2000 

eth1       Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:531 errors:0 dropped:0 overruns:0 frame:12602
           TX packets:1831 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000 
           RX bytes:85048 (83.0 KiB)  TX bytes:375706 (366.9 KiB)
           Interrupt:3 Base address:0x8000 

eth2       Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:0 errors:0 dropped:0 overruns:0 frame:0
           TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000 
           RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
           Interrupt:5 Base address:0x8000 

lo         Link encap:Local Loopback  
           inet addr:127.0.0.1  Mask:255.0.0.0
           UP LOOPBACK RUNNING MULTICAST  MTU:16436  Metric:1
           RX packets:50 errors:0 dropped:0 overruns:0 frame:0
           TX packets:50 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:7620 (7.4 KiB)  TX bytes:7620 (7.4 KiB)

vlan1      Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:6586 errors:0 dropped:0 overruns:0 frame:0
           TX packets:8057 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:1105333 (1.0 MiB)  TX bytes:4017441 (3.8 MiB)

vlan100    Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           inet addr:YOUR.PUBLIC.IP  Bcast:YOUR.BCAST.IP  Mask:255.255.128.0
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:1400 errors:0 dropped:0 overruns:0 frame:0
           TX packets:1308 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:270035 (263.7 KiB)  TX bytes:141600 (138.2 KiB)

vlan101    Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:0 errors:0 dropped:0 overruns:0 frame:0
           TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

vlan105    Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:0 errors:0 dropped:0 overruns:0 frame:0
           TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0 
           RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

wl0.1      Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:0 errors:0 dropped:0 overruns:0 frame:12602
           TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:1000 
           RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

Now that the VLAN's are created, we just need to create the trunk port on the router:
```
filipe@RT-AC66U:/tmp/home/root# robocfg vlan 1 ports "1 2 3 8t"
filipe@RT-AC66U:/tmp/home/root# robocfg vlan 100 ports "0t 4t 8t"
filipe@RT-AC66U:/tmp/home/root# robocfg vlan 101 ports "0t 4t 8t"
filipe@RT-AC66U:/tmp/home/root# robocfg vlan 105 ports "0t 4t 8t"
```

So first we are removing the port 4 from the default VLAN1. 

Then we are adding the VLANs 100, 101 and 105 to the ports "0t 4t 8t". 
Essentially we are telling the router to duplicate the trunked network on the WAN port 0 
to the LAN port 4 in addition to itself via port 8. 

This port 8 is a special port (called the CPU internal port). If you want the router to interact 
with any of the network traffic on the ports, it needs to be "plugged into" the CPU internal port. 
If omitted, the router will pass along the traffic from one external port to another and otherwise not 
pay attention to it.

Also, the "t" and some other letters stand for:

| Letter  | Functionality        |   
| --------|:--------------------:| 
| t       | Tagged               | 
| u       | Untagged             | 
| *       | CPU internal default |

**Note that some routers have the ports inverted.** (Asus RT-N16 and RT-AC87U might suffer from this)  
If this happens, you will need to trunk with "1t" instead of the "4t", and this will result on a port 4 trunk.

Also, I'm using **robocfg** that is the **switch configuration utility**. 
But note that these commands could also be done with **nvram** (non-volatile RAM) but
the name would hint that the commands would persist at reboot, **but this is not the case**, 
it's designed to reset to defaults at reboot.

Finnaly if we use **robocfg** again (MACs replaced):
```
filipe@RT-AC66U:/tmp/home/root# robocfg show
Switch: enabled gigabit
Port 0: 1000FD enabled stp: none vlan: 2 jumbo: off mac: 00:00:00:00:00:00
Port 1: 1000FD enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
Port 2:   DOWN enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
Port 3:   DOWN enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
Port 4: 1000FD enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
Port 8: 1000FD enabled stp: none vlan: 1 jumbo: off mac: 00:00:00:00:00:00
VLANs: BCM53115 enabled mac_check mac_hash
   1: vlan1: 1 2 3 8t
   2: vlan2: 0 8u
 100: vlan100: 0t 4t 8t
 101: vlan101: 0t 4t 8t
 105: vlan105: 0t 4t 8t
```

#### Create the script

At this point everything should be working, but like I said, once we reboot the settings will be gone.  
To avoid this, we need to create a script with the previous commands and place them on the JFFS partition.

We can do this manually with the default **vi** editor:  
`vi /jffs/scripts/services-start`

Type `i` to enter edit mode, then paste the code bellow with your SSH client paste shortcut.  
Press `esc` to exit edit mode, type `:` to enter command mode, and type `wq` to save and exit.

Also, note that the script has to be called **services-start** so that Merlin can invoke it.  
[Refer to the documentation for details.](https://github.com/RMerl/asuswrt-merlin/wiki/User-scripts)

```bash
#!/bin/sh
logger "Starting configuration of VLans 100 (NET) 101 (VOIP) and 105 (IPTV)"

logger "Creating VLans 101 and 105"
vconfig add eth0 101
vconfig add eth0 105

logger "Enabling VLans 101 and 105"
ifconfig vlan101 up
ifconfig vlan105 up

logger "Creating trunk port 4 for VLans 100, 101 and 105"
robocfg vlan 1 ports "1 2 3 8t"
robocfg vlan 100 ports "0t 4t 8t"
robocfg vlan 101 ports "0t 4t 8t"
robocfg vlan 105 ports "0t 4t 8t"

logger "Done. Secondary router should be functional."
```

Finally we just need to make the script executable:  
`chmod a+rx /jffs/scripts/services-start`

Now if you reboot the router the settings will be applied at boot.  
You should see the `logger` messages from the script on:
* Advanced Settings → System Log → General Log
