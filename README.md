
Raspberry pi as wifi repeater
===============================

This is a simple guide to set Raspberry pi as wifi repeater, i know there are many guides out there, [including the official](https://github.com/raspberrypi/documentation/blob/master/configuration/wireless/access-point-routed.md), but none of them worked 100% without modifications.

Based on: `Linux raspberrypi 4.9.80-v7+`

In order to achieve our goal we will need to setup two separate interfaces,
You will need additional wifi usb device or use the ethernet connection.

Lets assume your home lan address is 10.0.0.0/24,
We will extend this network using another lan at address 10.0.1.0/24.

I'll assume the OS is up and running, and the wifi\ethernet interface to the router "10.0.0.0/24" is set up, .

```
                                               this interface is being configured
                                                    (10.0.1.0/24 network)
# Option 1                                                    |
                                                              |
                 +- Router ----+          +--- Raspberry ---+ /        +- Laptop ----+
                 | DHCP server |          | 10.0.0.18       |/         | WLAN Client |
(Internet)---WAN-+             +---LAN----|/        WLAN AP +-)))  (((-+             |
                 | 10.0.0.0/24 |          |        10.0.1.1 |          | 10.0.1.36   |
                 +-------------+          +-----------------+          +-------------+


# Option 2
                 +- Router ----+          +--- Raspberry ---+          +- Laptop ----+
                 | DHCP server |   WiFi   | 10.0.0.18       |          | WLAN Client |
(Internet)---WAN-+             +-)))  (((-|/        WLAN AP +-)))  (((-+             |
                 | 10.0.0.0/24 |          |        10.0.1.1 |          | 10.0.1.36   |
                 +-------------+          +-----------------+          +-------------+
```



Automatic Installing via Ansible:
-----------------------
Execute from project root directory  
(note for the `,` below, or the ip/dns will be considered a hosts inventory filename)
```bash
ansible-playbook -u pi --ask-pass \
    -i "[RASPBERRY_PI/DNS]," \
    ansible/setup_repeater.yaml
```

install example with custom values,  
the configured device ip from above example is `10.0.0.18`
```bash
ansible-playbook -u pi --ask-pass \
    -i "10.0.0.18," \
    ansible/setup_repeater.yaml \
    -e ap_ssid_name=SSID \
    -e ap_ssid_pass=PASSWORD
```

* default password for `pi` user is: `raspberry`

Manual Installation:
--------------------
When doing `ip a` you should have 4\3 interfaces,
loopback, eth0 for lan cable connection, wlan0 is the onboard wifi
and wlan1 is the usb wifi interface if connected.

wlan1 or eth0 will connect to your home LAN at 10.0.0.0/24  
wlan0 will create new network using 10.0.1.0/24
then a bridge (br0) will bridge the interfaces.

first provide your home SSID and password `sudo vi /etc/wpa_supplicant/wpa_supplicant.conf`
```
network={
    ssid="YOUR_HOME_SSID"
    psk="WIFI_PASSWORD"
}
```

Once raspberry was able to connect to the router (with internet access), do
```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install dnsmasq hostapd bridge-utils
```

Now configure static IP for wlan1 and static IP for wlan0 `sudo vi /etc/dhcpcd.conf`
Add to the end of file lines:
```bash
## If using additional wifi
interface wlan1
    static ip_address=10.0.0.2/24
    static routers=10.0.0.1
    static domain_name_servers=10.0.0.1 8.8.8.8

## If using ethernet
#interface eth0
#    static ip_address=10.0.0.2/24
#    static routers=10.0.0.1
#    static domain_name_servers=10.0.0.1 8.8.8.8

interface wlan0
    static ip_address=10.0.1.1/24
```

Restart the service `sudo service dhcpcd restart`

Set up the DHCP server (dnsmasq)
--------------------------------
```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig  
sudo vi /etc/dnsmasq.conf
```
And append
```bash
interface=wlan0
    dhcp-range=10.0.1.2,10.0.1.100,255.255.255.0,12h
```


Configure Access Point (hostapd)
--------------------------------
type `sudo vi /etc/hostapd/hostapd.conf`
And append
```bash
interface=wlan0
driver=nl80211

ssid=[AP_SSID]
hw_mode=g
channel=6
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=[AP_PASS]
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```
Save and exit, then `sudo vi /etc/default/hostapd` and replace line #DAEMON_CONF with
`DAEMON_CONF="/etc/hostapd/hostapd.conf"`

Start the services
```bash
sudo systemctl start hostapd
sudo systemctl start dnsmasq
```

Add Routing
-----------
type `sudo vi /etc/sysctl.conf` and uncommon line `net.ipv4.ip_forward=1`
Then run
```bash
sudo iptables -t nat -A  POSTROUTING -o wan1 -j MASQUERADE
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

Edit `sudo vi /etc/rc.local` and append above "exit 0" line 
```bash
iptables-restore < /etc/iptables.ipv4.nat
```

Then `sudo reboot`

When device boots up run
```bash
sudo brctl addbr br0

## If using wlan0
sudo brctl addif br0 wlan1

## If using eth0
#sudo brctl addif br0 eth0
```
