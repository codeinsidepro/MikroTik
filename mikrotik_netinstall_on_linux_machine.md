
## MikroTik Netinstall on Linux.

|                         |                                                             |
| ----------------------- | ----------------------------------------------------------- |
| **Author:**             | SM Wahiduzzaman                                             |
| **Mobile:**             | +8801823 555 556                                            |
| **Skype:**              | live:.cid.8b4a930feae477f8                                  |
| **Telegram:**           | https://t.me/codeinsidepro                                  |
| **E-Mail:**             | [codeinside.pro@gmail.com](mailto:codeinside.pro@gmail.com) |
| **Warranty/Guarantee:** | No                                                          |
| **Last Update:**        | 25-mar-2024                                                 |
|                         |                                                             |
|                         |                                                             |


I’m writing this because the MikroTik [official docs](https://help.mikrotik.com/docs/display/ROS/Netinstall) are thin on details for this use case.

`NetInstall` needs to force I/O through a single network path under all conditions in order to do what it does. This might seem like an easy thing to accomplish, but then realize that `NetInstall` operates at a very low level, and there are multiple stages to the conversation, each of which may have different rules applied by the OS’s network stack.

This recommendation holds even for those running Linux natively on the host system. While you can run `netinstall-cli` directly in that case.

The only key configuration choice is `bridging` the virtual network adapter to the one-and-only host-side Ethernet adapter that `netinstall-cli` will communicate over. Success lies in avoiding cleverness like NAT, “shared” networking, automatic switching between Ethernet and WiFi, etc.

Here is the documents what worked for me with Fedora Linux 39. Almost same for for other Linux distro.

### Firewall
---
The key server-side change is that many Linux OSes ship with a firewall enabled which will block the ports `netinstall-cli` needs when communicating with the router. 

The tricky bit is, the minimum set of ports isn’t documented anywhere, that I can see. 

Red Hat OS use [`firewalld`](https://firewalld.org/) these days, where the commands to unblock the required ports are below.
```bash
sudo firewall-cmd --add-port bootps/udp
sudo firewall-cmd --add-port tftp/udp
sudo firewall-cmd --add-port 5000/udp
```

Other Linuxes use other firewall systems. Some still use raw `iptables` or `nft` commands, `ufw` is popular on Ubuntu, etc.

Alternatively you can stop Firewall during installation.


### Disable `tftp` and port `udp/69`
---
You may get error message `bind tftp general failed: Address in use` indicates that the TFTP (Trivial File Transfer Protocol) port is already in use by another process on your system. TFTP typically operates on `UDP` port `69`.

To resolve this issue, you'll need to identify the process that is currently using the TFTP port and either stop that process or configure it to use a different port.

Find out which process is using the TFTP port
```bash
sudo netstat -tuln | grep :69
```
Output:
```
udp6       0      0 :::69                   :::*                               
```
This command will show you the process ID (PID) of the process using port 69. 

Once you have identified the PID, you can use the following command to find out more details about the process.
```bash
sudo lsof -i UDP:69
```
Output:
```
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
systemd   1 root  303u  IPv6    795      0t0  UDP *:tftp 
```


This will give you information about the process and its associated files. Once you know which process is using the TFTP port, you can decide how to proceed.
If the process is not necessary or can be stopped temporarily, you can stop it using:
```bash
sudo systemctl stop <process-name>
```

If you get systemd is using the TFTP port (UDP port 69). systemd is a system and service manager for Linux operating systems. To resolve the port conflict, you have a couple of options

Or
You can also stop the systemd TFTP service temporarily to free up the port. Use the following command:
```bash
sudo systemctl stop tftp
```
Output:
```
Stopping 'tftp.service', but its triggering units are still active:
tftp.socket
```

Check `tftp` status
```bash
sudo systemctl status tftp
```


### Download `netinstall`
---
The Linux version is a command line tool, which offers nearly the same parameters as the Windows counterpart. 

[Download](https://mikrotik.com/download) and save MikroTik ROS package `*.npk` and `netinstall-cli` in same directory in your Linux machine, Make sure both `*.npk` and `netinstall-cli`  files are same version.

I don’t know how critical it is to use the matching version of `netinstall-cli` when changing RouterOS versions, but while you’re downloading fresh `*.npk` , you might as well update `netinstall-cli` version as well.

Download the tool from our download page (links not literal)
```
wget https://download.mikrotik.com/routeros/[VERSION]/netinstall-[VERSION].tar.gz
```

Extract it
```
tar -xzf netinstall-[VERSION].tar.gz
```


### Networking
---
Change your Linux machine/Netinstall server’s IP address to static IP, use the `192.168.88.1/24`, Gateway is not require in our case.

Linux Client machine: 192.168.88.1/24

Only one Ethernet port on your router will participate in an `EtherBoot` conversation. It might be marked `BOOT` on your Router, but if not, it’s generally the one that comes up as `ether1` in the default configuration. 

The first two required ports aren’t much of a surprise given the mention of “BOOTP” in the official docs, but I had to do a packet capture to work out that the last one was required. Without it, you’ll get stuck at the “`sendFile`” step. It's better to disable your WiFi.

`NetInstall` will get stuck in the “`Waiting for RouterBOARD...`” step if you have the Ethernet cable plugged into the wrong port.


Check Linux machine/Netinstall server interface.
```shell
ip a | grep enp
```
Output:
```
2: enp4s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000

```

Or, Check Linux machine/Netinstall server interface.
```shell
ifconfig | grep enp
```
Output:
```
enp4s0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
```

The `enp4s0` value will vary by OS and virtual hardware configuration. On modern Linux OS, say “`ip link`” to get a list of possible names. For a VM, there are likely only two; pick the one that _isn’t_ the `lo` interface.


### Physical connection
---

- Use a UTP/STP cable to physically connect between the Linux Client machine and Router

- Connect the router Ether Port(`Eth1`/`BOOT`) directly to the `Netinstall` server’s/Linux Machine's copper Ethernet port; there merely needs to be an unimpeded L2 path between the two devices. 

### Netinstall Server Configuration(Linux machine)
---
You can follow below syntax to run command.

`netinstall-cli` command syntax.
```syntax

Version: 7.13.1(2024-01-05 14:35:24)
Usage: ./netinstall-cli [-r] [-e] [-b] [-k <keyfile>] [-s <userscript>] {-i <interface> | -a <client-ip>} [PACKAGE]+
	-r  apply default configuration
	-e  apply empty configuration
	    -r and -e are mutually exclusive
	    by default existing configuration will be kept
	-b  remove branding

```


**Now you can start the `netinstall-cli`server:**

Put all files `netinstall-cli` and `*.npk` in same directory.

List all files
```bash
ls -lha
```

Open the directory where you saved `netinstall-cli`> Click right button> `Open in Terminal`> Now run below command.
```bash
sudo chmod +x ./netinstall-cli
```

Run below command to install `arm` package
```bash
sudo ./netinstall-cli -r -i enp4s0 -a 192.168.88.2 routeros-7.13.1-arm.npk
```
Output:
```
routeros-7.13.1-arm.npk 
Version: 7.13.1(2024-01-05 14:35:24)
Will reset to default config
Using Interface: enp4s0
Wait for Link-UP on 'enp4s0'. OK
Using Client IP: 192.168.88.2
Using Server IP: 192.168.88.1
Starting PXE server
Waiting for RouterBOARD...
client: 74:4D:28:7B:7D:93
Detected client architecture: arm
Sending and starting Netinstall boot image ... 
Installed branding package detected
Discovered RouterBOARD...
Formatting...
Sending package routeros-7.13.1-arm.npk ...
Ready for reboot...
Sent reboot command

```

Or
Run below command to install `tile` package
```bash
sudo ./netinstall-cli -r -i enp4s0 -a 192.168.88.2 routeros-tile-6.49.10.npk 
```
Output:
```
Version: 7.14.1(2024-03-08 13:35:10)
Will reset to default config
Using Interface: enp4s0
Wait for Link-UP on 'enp4s0'............ OK
Using Client IP: 192.168.88.2
Using Server IP: 192.168.88.1
Starting PXE server
Waiting for RouterBOARD...
client: 08:55:31:B4:3D:1F
Detected client architecture: tile
Sending and starting Netinstall boot image ... 
Installed branding package detected
Discovered RouterBOARD...
Formatting...
Sending package routeros-tile-6.49.10.npk ...
Ready for reboot...
Sent reboot command
wahid@wahid-lenovo:/run/media/wahid/DATA/OFFICIAL DATA/MyCloud Offline/MikroTik/Router/ROS/Long-Term 6.49.10/netinstall-7.14.1$ 
```


**Now the important part you have to do:**
- Remove power cord of your Router, Then press reset button in your Router, in the mean time plugin power cord again, Wait for approximately 30 sec, release reset button. You will see above output in command line. If you see above output that means `RouterOS` installation done successfully!!!

- For routers with wired interfaces only, the base `routeros-*.npk` package is all you require, but for WiFi based routers, if you fail to at least include the appropriate wireless package, the default configuration is likely to come up improperly. Anything else you add `syntax` to this is purely optional.

- If you get the `Key was rejected` message, hit Ctrl-C to break out of `netinstall-cli`, then Up-Arrow and Enter to quickly restart it. I’ve seen this bypass the symptom when using a CentOS 8 Stream VM as the server.

> **Note:** 
> I highly recommend running a packet sniffer like Wireshark or tcpdump when you’re doing this, to identify any configuration errors.

