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
```bash
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
From here on, you should not have to use the `-o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false` flags.
### Starting WPA Supplicant per default
You need to create two files. First `/opt/wpa-supplicant.sh`.
```bash
#!/bin/sh
/sbin/wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa-suppplicant.conf
```
And set its rights:
```
# chmod +x /opt/wpa-supplicant.sh
```
Then create `/usr/lib/systemd/wpa.service` with the following content:
```
[Unit]
Description=WPA Supplicant initialisation

[Service]
Type=idle
ExecStart=/opt/wpa-supplicant.sh

[Install]
WantedBy=multi-user.target
```
And finally start it on boot:
```
# systemctl enable wpa.service
```
That should mean your WPA Supplicant now starts per default. Reboot to check.
### Setting upp DHCP
```bash
# apt -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false dnsmasq

```





```bash
#!/bin/sh
/sbin/wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa-suppplicant.conf
```





20211124_raspi_4_bookworm.img.xz






