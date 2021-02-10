# raspberry-pi-router

## How to configure a Raspberry Pi 4 as a home network router



## Hardware

Required:
- Raspberry Pi 4
- TP-LINK USB 3.0 Gigabit Ethernet adapter UE300
- microSD card used as the boot disk (minimum capacity 8 GB)
- Power source

Useful when setting up or troubleshooting:
- Monitor with a micro HDMI cable
- USB keyboard

### Initialize Boot disk 

1. Download `Raspberry Pi OS Lite` from the [Raspberry Pi website](https://www.raspberrypi.org/software/operating-systems/).

1. Install the image `raspbian_lite_latest.zip` on a microSD memory card with [balenaEtcher](https://www.balena.io/etcher/).


### Connect peripherals

1. Plug in the memory card to the microSD card slot. It is located on the bottom side of the Raspberry Pi at the end opposite to the USB connectors.

1. Plug in the Ethernet adapter to a blue USB 3.0 connector,

1. Plug in a monitor with a micro HDMI cable to either micro HDMI port.

1. Plug in a USB keyboard to any available USB port.

1. Plug in a power source to the USB-C connector.  


## Software

### Raspberry Pi Software Configuration

Start the Raspberry Pi Software Configuration Tool `raspi-config`:
```
$ sudo raspi-config
```

Select the following options:

5\. Localisation Options
  - L3 Keyboard
    - **select the proper keyboard layout**
  - L2 Timezone
    - **select the proper timezone**
  - L4 WLAN Country
    - **select the proper WLAN country**

1\. System Options
  - S3 Password
    - **select a new password** for the `pi` user
  - S4 Hostname
    - **select a new hostname** for the computer

3\. Interface Options
  - P2 SSH
    - **enable** remote command line access using SSH

6\. Advanced Options
  - A1 Expand Filesystem
    ** select this** to use all available storage space on your boot media
  - A4 Network Interface Names
    - **enable** predictable network interface names
    - this will make a USB ethernet adapter with MAC address `d0:37:45:a7:7f:a3` appear as interface ´enxd03745a77fa3`

### Upgrade and install packages

Upgrade packages to latest versions.

```
$ sudo apt update && sudo apt upgrade 

```

Install the following packages to run a DNS and DHCP server on the intranet and save and restore firewall rules:

```
$ sudo apt install dnsmasq dnsutils netfilter-persistent iptables-persistent screen
```

`dnsutils` contains the command `dig` that is useful for checking that domain name service queries work both for local and Internet addresses.   

`screen` is useful for persisting a terminal connection.

### Configure WiFi access point

TODO

### Enable IP forwarding 

Uncomment the following line on `/etc/sysctl.conf` or add file `/etc/sysctl.d/ip-routing.conf` with the following content:
```
net.ipv4.ip_forward=1
```

Then run the following command to apply the new setting without rebooting: 

```
$ sudo sysctl --system
```

### Enable masquerading

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

### Configure local area network 

Find out name of the external ethernet adapter with this command, which lists all network interfaces:

```
$ ip a
...
3: enxd03745a77fa3: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
...
```

The name should have prefix `enx`. In the example above the interface name is `enxd03745a77fa3`

Append DHCP client configuration to file `/etc/dhcpcd.conf`:

```
interface eth0
  static domain_name=home

# TODO use name of your interface
interface enxd03745a77fa3
  static ip_address=10.10.0.1/24

interface wlan0
  static ip_address=10.10.0.1/24
  nohook wpa_supplicant
```

Append DNS and DHCP server configuration to file `/etc/dnsmasq.conf`:

```
# run DNS and DHCP server on these interfaces
# TODO use name of your interface
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

## Getting it running

When all configuration is done, do the following:
 
1. Power off Raspberry Pi
1. Plug in Internet ethernet cable to internal ethernet port
1. Plug in local area network (LAN) ethernet cable to external ethernet adapter 
1. Power on Raspberry Pi

Devices connected to LAN should get local IP addresses in the range `10.0.0.1/24` from the DHCP server on the Raspberry Pi and be able to connect the Internet.


## Resources

- [[SOLVED] r8153 USB Network Device shows up as CD-ROM on boot](https://bbs.archlinux.org/viewtopic.php?id=228195)
- [Setting up a Raspberry Pi as a routed wireless access point
](https://www.raspberrypi.org/documentation/configuration/wireless/access-point-routed.md)