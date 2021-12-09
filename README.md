# Using a Raspberry Pi as a WiFi dongle
This will be based in large parts on [Debian for Raspberry Pi](https://wiki.debian.org/RaspberryPi).
Using a 4B, 400 or a CM4, the bootup is based on a UEFI compatible setup. The others need [a binary blob bootloader](https://packages.debian.org/search?keywords=raspi-firmware). 
## Preparations
### Prepare your machine
You might have to update your firmware, see https://github.com/raspberrypi/rpi-eeprom.
### Prepare the target Debian install
From another linux installation:
1. Download the Debian Raspberry Pi image for your version: https://raspi.debian.net/tested-images/
2. Burn to storage device, as root
```
# xzcat -c «distroimage» | pv | tee «targetdevice» > /dev/null
```
or as a sudo capable user:
```
$ xzcat -c «distroimage» | pv | sudo tee «targetdevice» > /dev/null
```
Note: `| pv` is just for viewing progress of writing the image, and might not even be installed on the system of origin, so you can remove that part if you don't want it.
## On the machine
Assuming you are the root user (default Debian does not have a set password for root).
### Setting up WiFi
First we need to get the wireless set up to receive DHCP.
```
# echo "allow-hotplug wlan0" >> /etc/network/interfaces.d/wlan0
# echo "iface wlan0 inet dhcp" >> /etc/network/interfaces.d/wlan0
```
However, if you instead need a static IP, the syntax would be 
```
auto wlan0
iface wlan0 inet static
address «ipv4address»
netmask «ipv4netmask»
gateway «ipv4gatewayip»
dns-nameservers «ipv4dnsip»
```
Anyway, the next step would be to install and configure the WPA Supplicant:
```
# apt install wpasupplicant
```
After this we need to configure the WPA supplicant. If you have a regular home network, this would look like:
```
# echo "ctrl_interface=/var/run/wpa_supplicant" > /etc/wpa_supplicant/wpa_supplicant.conf
# wpa_passphrase "«WiFiSSID»" "«WiFiPassphrase»" | tee -a /etc/wpa_supplicant/wpa_supplicant.conf
```
For an Enterprise type WiFi network, this would depend a lot on how it's set up. In the case of eduroam for MIUN, it would look something like this, though I haven't been able to verify this:
```
network={
	scan_ssid=1
	ssid="eduroam"
	key_mgmt=WPA_EAP
	eap=PEAP
	identity="«studentmail»"
	password="«password»"
	#phase1="peapver=0"
	#phase1="peaplabel=auto peapver=0"
	phase2="auth=MSCHAPv2"
}
```
Now it's time to reboot and once booted start the WPA Supplicant:
```
# /sbin/wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa-suppplicant.conf
```
### Updating the system
You should now have internet through WiFi, which means it is time to upgrade and update. However, the Raspberry Pi lacks a hardware clock and will take time from the boot imnage instead, and the `apt` package manager will not allow installing from packages set in the future from the system time as default. Therefore you need to run:
```
# apt -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false update
# apt -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false upgrade
```
### Localising the system
```
# apt -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false install keyboard-configuration
# apt -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false install console-setup
# apt -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false install ntp
```
Here, use [Debian:Keyboard](https://wiki.debian.org/Keyboard) for reference on how to setup your keyboard, [Debian:Locale](https://wiki.debian.org/Locale) for your language, and [Debian:DateTime](https://wiki.debian.org/DateTime) for reference on how to set up your time, date and timezone settings.
From here on, you should not have to use the `-o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false` flags to `apt` any longer..
### Starting WPA Supplicant per default
You need to create two files. First `/opt/wpa-supplicant.sh`.
```bash
#!/bin/bash
/sbin/wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa-suppplicant.conf
```
And set its rights:
```
# chmod +x /opt/wpa-supplicant.sh
```
Then create `/usr/lib/systemd/system/wpa.service` with the following content:
```
[Unit]
Description=WPA Supplicant Initialisation
After=wpa_supplicant.service

[Service]
Type=idle
ExecStart=/usr/bin/bash /opt/wpa-supplicant.sh

[Install]
WantedBy=multi-user.target
```
And finally start it on boot:
```
# systemctl enable wpa.service
```
That should mean your WPA Supplicant now starts per default. Reboot to check.
### Setting upp a network bridge
Debian has a nice page about this at [Debian:Bridge Network Connections](https://wiki.debian.org/BridgeNetworkConnections). 
```
# apt install bridge-utils
# brctl addbr br0
# brctl addif br0 eth0 wlan0
```
And as per the Debian page, setting up the DHCP




### Setting up DHCP
```
# apt install dnsmasq
```





Debian
https://wiki.debian.org/RaspberryPi4



https://linuxhint.com/debian_etc_network_interfaces/

Bridge
https://pimylifeup.com/raspberry-pi-wifi-bridge/
https://stackoverflow.com/questions/19815844/dnsmasq-on-an-interface-connected-to-a-linux-bridge-not-working
https://wiki.debian.org/BridgeNetworkConnectionsProxyArp
https://wiki.debian.org/BridgeNetworkConnections
https://raspberrypi.stackexchange.com/questions/34968/bridge-eth0-and-eth1-and-run-dnsmasq
https://www.elementzonline.com/blog/sharing-or-bridging-internet-to-ethernet-from-wifi-raspberry-pI
https://www.instructables.com/Raspberry-Pi-Ethernet-to-Wifi-Bridge/
https://willhaley.com/blog/raspberry-pi-wifi-ethernet-bridge/
https://superuser.com/questions/1540424/what-are-the-real-world-differences-between-bind-systemd-resolved-dnsmasq-etc

WiFi
https://www.miun.se/medarbetare/gemensamt/servicetjanster/it/natverk/
https://manpages.debian.org/stretch/wpasupplicant/wpa_supplicant.conf.5.en.html
https://unix.stackexchange.com/questions/367277/configure-wireless-interface-for-multiple-locations
https://linuxconfig.org/etcnetworkinterfacesto-connect-ubuntu-to-a-wireless-network
https://askubuntu.com/questions/497540/where-is-my-wpa-supplicant-conf
https://raspberrypi.stackexchange.com/questions/85599/how-to-start-stop-wpa-supplicant-on-default-raspbian
https://unix.stackexchange.com/questions/182847/how-to-automatically-apply-wpa-supplicant-configuration
https://www.linuxquestions.org/questions/linux-wireless-networking-41/wlanconnection-wpa2-eap-peap%3Bmschapv2-how-to-configure-633456/
https://www.linuxbabe.com/command-line/ubuntu-server-16-04-wifi-wpa-supplicant
