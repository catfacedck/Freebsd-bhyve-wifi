# Freebsd-bhyve WiFi on host laptop

Guest with WiFi host networking that uses pf.
In order to support WiFi from the host, provisioning is a bit more complicated than when using wired Ethernet on the host (see the handbook) as WiFi interfaces cannot support multiple IP addresses.

A. This WiFi freebsd 14-stable laptop host install tested with *BSD guests.
	
	- Freebsd host is assigned IP address 10.0.0.1/24 while guest is assigned IP address 10.0.0.2/24.

	- **Security Note: networking with laptop WiFi passed to the guest exposes your machine(s) on the Internet.**

	- vm-bhyve is not being used here and neither are zfs disks.

**_Step-by step procedure, assuming your freebsd host WiFi is wlan1:_**

1) Modify /etc/rc.conf, add the lines:
```
wlans_iwlwifi0="wlan1"
ifconfig_wlan1="WPA DHCP"

pf_enable="YES"
pf_rules="/etc/pf.conf" 
pflog_enable="YES"
pflog_logfile="/var/log/pflog"

gateway_enable="YES"
```

1a) Modify /boot/loader.conf to add the lines:
```
vmm_load=”YES” 
nmdm_load="YES"
```

2) Modify sysctl:
```
sysctl net.link.tap.up_on_open=1
```
(add permanently in /etc/sysctl.conf -> net.link.tap.up_on_open=1)

4) Add the following network taps on the host:
```
ifconfig bridge create name natif up
ifconfig tap0 create up 
ifconfig natif addm tap0
ifconfig natif inet 10.0.0.1/24
ifconfig tap0 up
```

6) Ensure Tiger VNC Viewer (pkg add tigervnc-viewer) or GTK-VNC Viewer (pkg add gtk-vnc) is installed.

7) To create disk0.img of size 20G:
```
truncate -s 20G disk0.img
```

9) Run your vm: (note - nvme only for SSD disk type, otherwise use virtio-blk, uses UEFI boot) Run -
```
ifconfig tap0 up
```
first.
```
bhyve -ADHP -c 4 -m 4G -w -H \
        -s 0,hostbridge \
        -s 4,nvme,disk0.img \
        -s 5,virtio-net,tap0 \
        -s 29,fbuf,tcp=0.0.0.0:5900,w=1400,h=900,wait \
        -s 31,lpc -l com1,stdio \
        -l bootrom,/usr/local/share/uefi-firmware/BHYVE_UEFI.fd \
        vm0
```
This assumes that you have previously installed your guest os from cdrom iso on disk0.img.
To do that add this line after the "hostbridge" line

```
-s 3,ahci-cd,/home/shingy/Downloads/FreeBSD-LATEST-ISO.iso \
```
Run the iso installer to install on disk0.img from inside your guest os.

7) Run TigerVNC viewer (if installed) from your applications menu or a shell as shown here:
``` 
   vncviewer -SendClipboard -AcceptClipboard -LowColorLevel -QualityLevel 6 0.0.0.0:5900 &
```

9) Provision the network on the guest:
```    
ifconfig vtnet0 inet 10.0.0.2/24 netmask 255.255.255.0
ifconfig route add default 10.0.0.1
```

Verify that all works:
Ping the freebsd bhyve host and the Internet: 
```
ping 10.0.0.1
ping www.yahoo.com
```

check with 
```
netstat -nr
```
```
Routing tables

Internet:
Destination        Gateway            Flags    Refs      Use  Netif Expire
default            10.0.0.1           UGSc        0        3 vtnet0       
10.0.0/24          link#1             UC          1        0 vtnet0       
10.0.0.1           link#1             UHLW        1        3 vtnet0       
127.0.0.1          127.0.0.1          UH          0        0    lo0  
```
check with 
```
ifconfig -a

vtnet0: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST> metric 0 mtu 1500
	options=28<VLAN_MTU,JUMBO_MTU>
	ether 00:a0:98:1b:c8:07
	inet6 fe80::2a0:98ff:fe1b:c807%vtnet0 prefixlen 64 scopeid 0x1
	inet 10.0.0.2 netmask 0xffffff00 broadcast 10.0.0.255
	media: Ethernet 1000baseT <full-duplex>
	status: active
 
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
	options=43<RXCSUM,TXCSUM,RSS>
	inet 127.0.0.1 netmask 0xff000000
	inet6 ::1 prefixlen 128
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x2
	groups: lo
```








   
