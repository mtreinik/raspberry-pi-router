# raspberry-pi-router

How to configure a Raspberry Pi 4 as a home network router

## Hardware

- Raspberry Pi 4
- TP-LINK USB 3.0 Gigabit Ethernet adapter UE300

## Software

### Packages to Install

Install the following packages to run a DNS and DHCP server on the intranet and save and restore firewall rules:

```
$ sudo apt install dnsmasq dnsutils netfilter-persistent iptables-persistent
```

`dnsutils` contains the command `dig` that is useful for checking that domain name service queries work both for local and Internet addresses.   

### Raspberry Pi Configuration

Start the Raspberry Pi Software Configuration Tool:
```
$ sudo raspi-config
```

Select the following options:

1\. System Options
  - S3 Password
    - **select a new password** for the `pi` user
  - S4 Hostname
    - **select a new hostname** for the computer

3\. Interface Options
  - P2 SSH
    - **enable** remote command line access using SSH

5\. Localisation Options
  - L2 Timezone
    - **select the proper timezone**

6\. Advanced Options
  - A4 Network Interface Names
    - **enable** predictable network interface names
    - this will make USB ethernet adapter with MAC address `d0:37:45:a7:7f:a3` appear as interface ´enxd03745a77fa3`

### Set up routing and IP masquerading

Uncomment the following line on `/etc/sysctl.conf` or add file `/etc/sysctl.d/ip-routing.conf` with the following content:
```
net.ipv4.ip_forward=1
```

Add a postrouting rule to table `nat` of iptables to masquerade traffic from other interfaces as coming from `eth0`: 

```
$ sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
$ sudo netfilter-persistent save
```

You can check the masquerading rule and its statistics with this command: 

```
$ sudo iptables -t nat -vL POSTROUTING
Chain POSTROUTING (policy ACCEPT 4984 packets, 331K bytes)
 pkts bytes target     prot opt in     out     source               destination         
 5169  441K MASQUERADE  all  --  any    eth0    anywhere             anywhere
```

Append DHCP client configuration to file `/etc/dhcpcd.conf`:

```
interface eth0
  static domain_name=home

interface enxd03745a77fa3
  static ip_address=10.10.0.1/24

interface wlan0
  static ip_address=10.100.0.1/24
  nohook wpa_supplicant
```

Append DNS and DHCP server configuration to file `/etc/dnsmasq.conf`:

```
# run DNS and DHCP server on these interfaces
interface=enxd03745a77fa3
interface=wlan0

# use local domain .home
domain=home
expand-hosts

# assign fixed IP to certain devices based on MAC address
dhcp-host=ac:dc:de:ad:be:ef,10.0.0.2

# give IP addresses from this range to other devices with lease time 24 hours
dhcp-range=10.0.0.1,10.0.0.250,24h
```

### Fix TP-LINK UE300 Mode

The particular USB 3.0 Ethernet adapter I used boots in USB Mass Media mode and needs to be reset before it works as an ethernet adapter. 

Reset the adapter by adding the following `udev` rule to file  `/etc/udev/rules.d/80-r8152.rules`:
```
ACTION=="add", SUBSYSTEM=="usb", ENV{ID_VENDOR_ID}=="2357", ENV{ID_MODEL_ID}=="0600", RUN+="/usr/sbin/usb_modeswitch -v 2357 -p 0600 -R"
```

## Misc Tips

The command `stty size` prints number of rows and columns of the terminal. Use these commands to reset terminal size to 80 columns by 24 rows: 
```
$ stty columns 80
$ stty rows 24
```   


## Resources

- [[SOLVED] r8153 USB Network Device shows up as CD-ROM on boot](https://bbs.archlinux.org/viewtopic.php?id=228195)
- [Setting up a Raspberry Pi as a routed wireless access point
](https://www.raspberrypi.org/documentation/configuration/wireless/access-point-routed.md)