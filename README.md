#RouterBerry Pi

** Turn your Raspberry Pi into a WiFi Router. This set of instructions will guide you to install and configure the packages needed to advertise it as a WiFi network and finally use it as a 2.4G WiFi router. **

* **[Requirements](#requirements)**
* **[Initial setup](#setup)**
* **[Configuring the DHCP server](#dhcp)**
* **[Configuring Hostapd](#hostapad)**
* **[Creating daemons to start the server up on reboot](#reboot)**

<a name="requirements"></a>

### Requirements

- A Raspberry Pi 3 Model B
- Latest version of Raspbian
- Internet access through Ethernet
- An optional [Wi-Fi Dual-Band WiFi USB Adapter](http://www.edimax.com/edimax/merchandise/merchandise_detail/data/edimax/in/wireless_adapters_ac600_dual-band/ew-7811uac/)

![](https://ezway-imagestore.s3.amazonaws.com/files/2018/01/4803116771515236624.png)

<a name="setup"></a>

### Initial setup

We will start by installing **hostapd** and **isc-dhcp-server** 

Log into the Raspberry Pi and in the terminal execute the following commands.
```bash
sudo apt-get update
sudo apt-get install hostapd isc-dhcp-server
```

<a name="dhcp"></a>

### Configuring the DHCP server
```bash
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.default
sudo vi /etc/dhcp/dhcpd.conf
```
Once dhcpd.conf opens, comment out the following lines
```bash
option domain-name "example.org";
option domain-name-servers ns1.example.org, ns2.example.org;
```
...now make sure this line is uncommented
```bash
authoritative;
```
...finally, depending on the amount of devices you want to accept connections from (**20 in our example**), we create a subnet by appending the following lines to the end of the file
```bash
subnet 192.168.42.0 netmask 255.255.255.0 {
    range 192.168.42.10 192.168.42.30;
    option broadcast-address 192.168.42.255;
    option routers 192.168.42.1;
    default-lease-time 600;
    max-lease-time 7200;
    option domain-name "local";
    option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```
Now we can specify what interface the DHCP server should use.

First we check the name of the interfaces by executing the following
```bash
ifconfig -a
```
This will return the name and addresses of the available interfaces, usually **eth0** for ethernet and **wlan0** for the wireless LAN.

Now we edit the DHCP server configuration file
```bash
sudo vi /etc/default/isc-dhcp-server
```
Once the file opens (and assuming the interface name was **wlan0**) add it by editing this line as follows
```bash
INTERFACES="wlan0"
```
We can now configure **wlan0** to use the static IP we configured in the subnet
```bash
sudo ifdown wlan0
sudo cp /etc/network/interfaces /etc/network/interfaces.backup
sudo vi /etc/network/interfaces
```
And edit the file to read as follows
```bash
source-directory /etc/network/interfaces.d
auto lo
iface lo inet loopback
iface eth0 inet dhcp
allow-hotplug wlan0
iface wlan0 inet static
  address 192.168.42.1
  netmask 255.255.255.0
  post-up iw dev $IFACE set power_save off
```
Save the file and start the interface on the static IP
```bash
sudo ifconfig wlan0 192.168.42.1
```

<a name="hostapad"></a>

### Configuring Hostapd
```bash
sudo vi /etc/hostapd/hostapd.conf
```
This file will setup the name of the wireless network and a password (if needed)
```bash
interface=wlan0
ssid=WIFI_DESIRED_NAME
hw_mode=g
channel=6
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=WIFI_DESIRED_PASSWORD
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```
We create a network translation between the ethernet port **eth0** and the wifi port **wlan0**
```bash
sudo cp /etc/sysctl.conf /etc/sysctl.conf.backup
sudo vi /etc/sysctl.conf
```
Once the file opens uncomment the following line
```bash
net.ipv4.ip_forward=1
```
...save the file and execute the following **immediately**
```bash
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```
We create iptables to set the translation
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
```
We make sure this configuration is set up on reboot
```bash
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
sudo vi /etc/network/interfaces
```
Once the file opens append the following line to the end
```bash
up iptables-restore < /etc/iptables.ipv4.nat
```

<a name="reboot"></a>

### Creating daemons to start the server up on reboot
```bash
sudo update-rc.d hostapd enable
sudo update-rc.d isc-dhcp-server enable
```
Now you can reboot the Pi to test it out
```bash
sudo reboot
```
