# RASPBERRY PI 3 - WIFI CLIENT STATION AND ACCESS POINT SIMULTANEOUSLY

Running the Raspberry Pi 3 Stretch as a WiFi Client (station) and Access Point (AP) from the single built-in wifi.

It's has been written about before, but this way is better. The access point device is created before networking
starts (using udev) and then, due to current OS bug, `/etc/rc.local` is need to starts the WiFi Client after the Access Point, otherwise this will not work and you will be left with dmesg outputs like this:

	brcmfmac: brcmf_c_set_joinpref_default: Set join_pref error (-1)
	brcmfmac: brcmf_cfg80211_connect: BRCMF_C_SET_SSID failed (-1)

## Configuring:

Here, we are using `uap0` as our Access Point interface name and `192.168.10.1` as our static ip address. Modify if you like but don't forget to do the same on the other files.

### /etc/network/interfaces.d/ap

	allow-hotplug uap0
	auto uap0
	iface uap0 inet static
	    address 192.168.10.1
	    netmask 255.255.255.0

### /etc/network/interfaces.d/station

	allow-hotplug wlan0
	iface wlan0 inet manual
	    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

### /etc/udev/rules.d/90-wireless.rules 

	ACTION=="add", SUBSYSTEM=="ieee80211", KERNEL=="phy0", \
	    RUN+="/sbin/iw phy %k interface add uap0 type __ap"

### Do NOT let DHCPCD manage wpa_supplicant!!

	rm -f /lib/dhcpcd/dhcpcd-hooks/10-wpa_supplicant

### OR if you don't want to delete it, simply change it's location like the command below:

	mv /lib/dhcpcd/dhcpcd-hooks/10-wpa_supplicant /home/pi/Desktop/10-wpa-supplicant-backup

Set up the client WiFi (station) on wlan0.
Create or modify `/etc/wpa_supplicant/wpa_supplicant.conf`.
The contents depend on whether your known networks are open, WEP or WPA.  It is probably WPA, and so should look like:

    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1
    country=BR
    
    network={
	    ssid="_NAME_SSID_"
	    psk=_PASSWORD_
	    key_mgmt=WPA-PSK
    }

Replace `_NAME_SSID_` with your known networks SSID (home, office, any network that you can access) and `_PASSWORD_` with your WiFi password (in clear text).

### Restart DHCPCD service:

	systemctl restart dhcpcd
	
### Bring up the station (Client) interface

	ifup wlan0
	
At this point your client WiFi should be connected.

### Manually invoke the udev rule for the AP interface.

Execute the command below. This will also bring up the `uap0` interface.  
It will wiggle the network, so you might be kicked off (esp. if you are logged into your Pi on wifi). Just log back on.

	/sbin/iw phy phy0 interface add uap0 type __ap
	
### Install the packages you need for DNS, Access Point and Firewall rules.

	apt-get update
	apt-get install hostapd dnsmasq

### /etc/dnsmasq.conf

	interface=lo,uap0
	no-dhcp-interface=lo,wlan0
	bind-interfaces
	server=8.8.8.8
	dhcp-range=192.168.10.50,192.168.10.150,12h

### /etc/hostapd/hostapd.conf

	interface=uap0
	ssid=_NAME_AP_SSID_
	hw_mode=g
	channel=11
	macaddr_acl=0
	auth_algs=1
	ignore_broadcast_ssid=0
	wpa=2
	wpa_passphrase=_AP_PASSWORD_
	wpa_key_mgmt=WPA-PSK
	wpa_pairwise=TKIP
	rsn_pairwise=CCMP

Replace `_NAME_AP_SSID_` with the SSID you want for your access point.  
Replace `_AP_PASSWORD_` with the password for your access point.
Make sure it has enough characters to be a legal password! (8 characters minimum).

### /etc/default/hostapd

	DAEMON_CONF="/etc/hostapd/hostapd.conf"

### Now restart the dnsmasq and hostapd services:

	systemctl restart dnsmasq
	systemctl restart hostapd

### Restart the client interface:

	ifdown wlan0
	ifup wlan0

Also, if the client interface went down for some reason (see above "bringup order").  Bring it back up.

### Permanently deal with interface bringup order:
See: https://unix.stackexchange.com/questions/396059/unable-to-establish-connection-with-mlme-connect-failed-ret-1-operation-not-p

This link is saying that you need to first load AP, wait a little, and then load the client, because of below:
> When both interfaces are brought up at the same time or close enough one to another, later interface tries to reuse already loaded instance of device driver, which is obviously impossible...

Edit `/etc/rc.local` and add the following lines just before "exit 0":

	sleep 5
	ifdown wlan0
	sleep 2
	rm -f /var/run/wpa_supplicant/wlan0
	ifup wlan0
	iptables -t nat -A POSTROUTING -s 192.168.2.0/24 ! -d 192.168.2.0.0/24 -j MASQUERADE

**(OPTIONAL 1)** Bridge AP to client side.
If you do this step, then someone connected to the AP side can browse the internet through the client side.

	echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
	echo 1 > /proc/sys/net/ipv4/ip_forward
	
If you can't execute the commands above because of permission denied, use the command below instead:

	sudo bash -c 'echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf'
	sudo bash -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'

**(OPTIONAL 2)** Also, if you use `raspberrypi.local` to connect to your board via VNC or similar, you can change its DNS name by editing the file on /etc/hosts:

	ifdown wlan0
	127.0.0.1       localhost
	::1             localhost ip6-localhost ip6-loopback
	ff02::1         ip6-allnodes
	ff02::2         ip6-allrouters
	
	127.0.1.1       raspberrypi
	_STATIC_IP_AP_    raspberrypi

Add the last line to your file and replace `_STATIC_IP_AP_` with your static ip AP chosen, the same one as the `/etc/network/interfaces.d/ap` file.

**(OPTIONAL 3)** Firewall rules (testing)

### Reboot your Pi board, just to be on the safe side and see if all the changes will persist.
