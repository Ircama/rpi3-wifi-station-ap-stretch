# RASPBERRY PI - WIFI STATION+AP

Running the Raspberry Pi 3 as a Wifi client (station) and access point (ap) from the single built-in wifi.

Its been written about before, but this way is better.  The access point device is created before networking
starts (using udev) and there is no need to run anything from `/etc/rc.local`.  No reboot, no scripts.

## Use Cases

The Rpi 3 wifi chipset can support running an access point and a station function simultaneously.

Also the Raspberry Pi Zero W accepts this procedure with the internal Wifi device, without external boards.

One use case is a device that connects to the cloud (the *station*, via a user's home wifi network) but
that needs an admin interface (the *access point*) to configure the network.  The user powers on the
device, then logs into the access point using a specified SSID/password.  The user runs a browser
and connects to the access point IP address (or hostname), which is running a web server to configure
the station network (the user's wifi).

Another use case might be to create a guest interface to your home wifi.  You can configure the client
side with your wifi particulars, then configure the access point with a password you can give out to your
guests.  When the party's over, change the access point password.

## /etc/network/interfaces.d/ap

```sh
sudo vi /etc/network/interfaces.d/ap
```

Then add these lines:

```sh
auto uap0
iface uap0 inet static
  address 192.168.50.1
  netmask 255.255.255.0
```

## /etc/network/interfaces

```sh
sudo vi /etc/network/interfaces
```

Then check this content:

```
# interfaces(5) file used by ifup(8) and ifdown(8)

# Please note that this file is written to be used with dhcpcd
# For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

allow-hotplug wlan0
iface wlan0 inet dhcp
  pre-up sleep 10
  wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

Note: *pre-up sleep 10* avoids the *linkdown* remark of *wlan0* in *net route* output.

## /etc/udev/rules.d/90-wireless.rules

```sh
sudo vi /etc/udev/rules.d/90-wireless.rules
```

Then add these lines:

```
ACTION=="add", SUBSYSTEM=="ieee80211", KERNEL=="phy0", \
    RUN+="/sbin/iw phy %k interface add uap0 type __ap"
```

## Manually invoke the rule.

Execute the command below. This will also bring up the uap0 interface.

```sh
sudo /sbin/iw phy phy0 interface add uap0 type __ap
```

Note: when executed for the second time (or if the above rule is already active), the command returns with the following error message:

    command failed: Device or resource busy (-16)

## Set up the client wifi (station) on wlan0.

Create `/etc/wpa_supplicant/wpa_supplicant.conf`.  The contents depend on whether your home network is open, WEP or WPA.  It is
probably WPA/WPA2 and so should look like:

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=US

network={
    ssid="_ST_SSID_"
    psk="_ST_PASSWORD_"
    key_mgmt=WPA-PSK
}

network={
    ssid="_ST_SSID2_"
    psk="_ST_PASSWORD2_"
    key_mgmt=WPA-PSK
}
```

Replace `_ST_SSID_` with your home network SSID and `_ST_PASSWORD_` with your wifi password (in clear text).
	
## Install the packages you need for DNS, Access Point and Firewall rules.

```sh
apt-get update
apt-get install -y hostapd dnsmasq iptables-persistent
```

## /etc/dnsmasq.conf

Add the following at the end of the file:

    interface=lo,uap0
    no-dhcp-interface=lo,wlan0
    bind-interfaces
    domain-needed
    bogus-priv
    server=8.8.8.8
    dhcp-range=192.168.50.50,192.168.50.255,12h

## /etc/hostapd/hostapd.conf

    interface=uap0
    ssid=_AP_SSID_
    hw_mode=g
    channel=7
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=_AP_PASSWORD_
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP

Replace `_AP_SSID_` with the SSID you want for your access point.  Replace `_AP_PASSWORD_` with the password for your access point.  Make sure it has
enough characters to be a legal password!  (8 characters minimum).

## /etc/default/hostapd

Add the following:

    DAEMON_CONF="/etc/hostapd/hostapd.conf"

## Bridge AP to client side

This is optional.  If you do this step, then someone connected to the AP side can browse the internet through the client side.

Edit */etc/sysctl.conf* and uncomment *net.ipv4.ip_forward=1*. Then:

```sh
sudo echo 1 > /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -A POSTROUTING -s 192.168.50.0/24 ! -d 192.168.50.0/24 -j MASQUERADE
sudo iptables-save > /etc/iptables/rules.v4
```

Example of output of *iptables-save*:

```
*filter
:INPUT ACCEPT [1479:117133]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [2332:441251]
COMMIT
*nat
:PREROUTING ACCEPT [56:9429]
:INPUT ACCEPT [54:8773]
:OUTPUT ACCEPT [22:1546]
:POSTROUTING ACCEPT [20:1387]
-A POSTROUTING -s 192.168.50.0/24 ! -d 192.168.50.0/24 -j MASQUERADE
COMMIT
```

## REBOOT!

    reboot

## Check configuration

Check routing table:

```sh
ip route
```

Valid output:

```
default via 192.168.1.1 dev wlan0
192.168.1.0/24 dev wlan0 proto kernel scope link src 192.168.1.140
192.168.50.0/24 dev uap0 proto kernel scope link src 192.168.50.1
```

Check that *linkdown* is not present.
Check that both *wlan0* and *uap0* are available, with valid IP interface and IP address ranges.

Check interface configuration:

```sh
ifconfig
```

Valid output:

```
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 27  bytes 1841 (1.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 27  bytes 1841 (1.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

uap0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.50.1  netmask 255.255.255.0  broadcast 192.168.50.255
        inet6 fe80::ba27:ebff:fee3:11ad  prefixlen 64  scopeid 0x20<link>
        ether b8:27:eb:e3:11:ad  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 31  bytes 4657 (4.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.140  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::ba27:ebff:fee3:11ad  prefixlen 64  scopeid 0x20<link>
        ether b8:27:eb:e3:11:ad  txqueuelen 1000  (Ethernet)
        RX packets 2014  bytes 174203 (170.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2768  bytes 611501 (597.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Check that both *wlan0* and *uap0* are available.
Check the *inet* IP addresses.
