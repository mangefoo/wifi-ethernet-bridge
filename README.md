Raspberry Pi WiFi-to-ethernet bridge
====================================

tl;dr
-----
- Install Raspbian on a Raspberry Pi and set up WiFi with SSH enabled
- Figure out the IP of the Raspberry Pi
- `$ export RPI_IP=<Newly found IP>`
- `$ ssh-copy-id pi@$RPI_IP`
- `$ ansible-playbook -i $RPI_IP, ansible/wifi-bridge.yml -u pi`

Background
----------
After changing internet provider our old fashion coax set-tob box was replaced by an IPTV box. The only connection type
available was ethernet and due to the placement of our TV we would have to draw a fairly long cable to connect to the
closest network switch. This and the fact that we were close to moving out of the apartment made me look for a different
solution. I didn't want to spend too much money on this since it was a temporary solution, so I discarded the option
of buying a commercial Wifi bridge product. I was also going to watch a game on TV the same night so getting it ready
quickly was a driver.

Solution
--------
I ended up going for an unused Raspberry pi 3 that I previously used as a [RetroPi](https://retropie.org.uk) for playing
old video games. It has integrated WiFi and ethernet port, which made it a perfect fit for this problem.

Setting it up as a Wifi bridge requires the following steps:
- [Configure WiFi on the wlan0 interface](#configure-wifi-on-the-wlan0-interface)
- [Set a static IP on the eth0 interface](#set-a-static-ip-on-the-eth0-interface)
- [Set up NAT forwarding on the eth0 interface](#set-up-nat-forwarding)
- [Turn on IP forwarding](#turn-on-ip-forwarding)
- [Set up a DHCP server on the eth0 interface](#set-up-a-dhcp-server)

### Configure WiFi on the wlan0 interface
This is done by updating the `/etc/wpa_supplicant/wpa_supplicant.conf` file. The content will vary between setups. This
is a typical configuration:
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=SE

network={
	ssid="<YOUR WIFI SSID>"
	psk="<YOUR WIFI PASSWORD>"
	key_mgmt=WPA-PSK
}
```
When setting up a new device you can put the `wpa_supplicant.conf` file in the `/boot` directory of the SD card. This
will then be copied to `/etc/wpa_supplicant.conf` on the first boot. This enables you to set up the device without any
keyboard or monitor. You can then log in through SSH and finalize the configuration. To enable SSH you will also need
 to create an empty `/boot/ssh` file on the SD card. The default username and password on Raspbian are ***pi*** and 
***raspberry***

### Set a static IP on the eth0 interface
Since we are the gateway and DHCP server of the wired network we need to set a static IP on the eterhnet interface. I
chose `192.168.44.0/24` as the network and `192.168.44.1` as the address of the Raspberry Pi. 

**dhcpcd** is used for network configuration in the Raspbian distribution, so we will use that to configure the network 
for the ethernet interface. This is done by adding or changing the `interface eth0` section in `/etc/dhcpcd.conf`:
```
interface eth0
static ip_address=192.168.44.1/24
```
On thing that made me supprised was that the interface was not configured until the ethernet cable was plugged in.
Running `ifconfig eth0` before the cable was plugged in showed no IP address on the interface, which made me waste some
time troubleshooting.

### Set up NAT forwarding
The IP tables configuration to turn on NAT forwarding in Linux is pretty straight forward. NAT forwarding is sometimes
called masquerading in IP tables terminology:
```
/sbin/iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
/sbin/iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
/sbin/iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
```
To make the configuration persistent through a reboot I installed the `iptables-persistent` package:
```
apt install iptables-persistent
```
If you have already set up the IP tables rules just answer 'Y' when the installer asks if you want to save the current
configuration. Otherwise, run `iptables-save` after the rules are set up. They will then be saved to `/etc/iptables` and
applied during boot.

### Turn on IP forwarding
By default, Linux installations does not forward packets between interfaces. This needs to be configured in order for it
to be able to act as a gateway. This can be done with:
```sysctl net.ipv4.ip_forward=1```
To make it persistent through reboots it should be added to `/etc/sysctl.conf`:
```
net.ipv4.ip_forward=1
```

### Set up a DHCP Server
I decided to use `dnsmasq` as a DHCP server since I hade previous experience with it. It is versatile and easy to set
up. I will also act as a DNS server for the clients in the network, hence the name. The default configuration contains
most that we need. Only some small changes are required:
- We need to make sure that we are only serving DHCP requests on the ethernet interface. Using the default values would
make the device reply to requests on the Wifi network as well which will cause problems. 
- We need to specify a network range within our 192.168.44.0/24 network that will be used for DHCP clients.
To do this add the following to `/etc/dnsmasq.conf`:
```
interface=eth0
dhcp-range=192.168.44.50,192.168.44.150,12h
```

