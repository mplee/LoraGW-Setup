# Lora Gateway base setup for SX1301 based concentrators

This setup is used for some LoraWAN concentrators based on small computers such as Raspberry PI or others. 
For example it works fine with the RAK831 PI Zero [shield](https://github.com/hallard/RAK831-Zero) 

<img src="https://raw.githubusercontent.com/hallard/RAK831-Zero/master/pictures/PiZero-RAK831-finished.jpg" alt="RAK831 Shield">     

And for the [iC880a](https://github.com/ch2i/iC880A-Raspberry-PI) sield for Raspberry PI V2 or V3.

<img src="https://raw.githubusercontent.com/ch2i/iC880A-Raspberry-PI/master/pictures/ic880a-mounted-V12.jpg" alt="iC880a Shield">     

# Installation

Download [Raspbian lite image](https://downloads.raspberrypi.org/raspbian_lite_latest) and [flash](https://www.raspberrypi.org/documentation/installation/installing-images/README.md) it to your SD card using [etcher](http://etcher.io/).

## Prepare SD to your environement

Once flashed, you need to do some changes on boot partition (windows users, remove and then replug SD card)

### Enable SSH

Create a dummy `ssh` file on this partition. By default SSH is now disabled so this is required to enable it. Windows users, make sure your file doesn't have an extension like .txt etc.

### Enable USB OTG (Pi Zero Only)

If you need to be able to use OTG (Console access for any computer by conecting the PI to computer USB port)
Open up the file `cmdline.txt`. Be careful with this file, it is very picky with its formatting! Each parameter is seperated by a single space (it does not use newlines). Insert `modules-load=dwc2,g_ether` after `rootwait quiet`.

The new file `cmdline.txt`  should looks like this
```
dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=PARTUUID=37665771-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait quiet modules-load=dwc2,g_ether init=/usr/lib/raspi-config/init_resize.sh
```

For OTG, add also the bottom of the `config.txt` file, on a new line 
```
dtoverlay=dwc2
```

### Optionnal, disable Auto Resize of SD Card 

And since I don't like the Auto Resize SD function (I prefer do do it manually from `raspi-config`), remove also from the file `cmdline.txt` auto resize by deleting the following 
```
init=/usr/lib/raspi-config/init_resize.sh
```

The new file `cmdline.txt`  should looks like this
```
dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=PARTUUID=37665771-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait quiet modules-load=dwc2,g_ether
```

### Pre Connect to your WiFi AP

Finally, on same partition (boot), to allow your PI to connect to your WiFi after first boot, create a file named `wpa_supplicant.conf` to allow the PI to be connected on your WiFi network.

``` 
country=FR
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
network={
  ssid="YOUR-WIFI-SSID"
  psk="YOUR-WIFI-PASSWORD"
}
``` 
Of course change country, ssid and psk with your own WiFi settings.


That's it, eject the SD card from your computer, put it in your Raspberry Pi Zero . It will take up to 90s to boot up (shorter on subsequent boots). You then can SSH into it using `raspberrypi.local` as the address.
If WiFi does not work, connect it via USB to your computer It should then appear as a USB Ethernet device.

```shell
ssh pi@raspberrypi.local
```

## Now connect to raspberry PI with ssh or via USB otg

Remember default login/paswword (ssh or serial console) is pi/raspberry.

So please **for security reasons, you should change this default password**
```shell
passwd 
``` 

```shell
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install git-core build-essential ntp scons python-dev swig python-psutil
``` 

If the 2nd command fire the following error (always happen with latest rasbian because it updates kernel and co) 
then just **reboot** and restart the 2 commands above.

```
Reading package lists... Done
Building dependency tree
Reading state information... Done
Package python-psutil is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source

Package ntp is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source

Package python-dev is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source
However the following packages replace it:
  python

E: Unable to locate package git-core
E: Package 'ntp' has no installation candidate
E: Unable to locate package scons
E: Package 'python-dev' has no installation candidate
E: Unable to locate package swig
E: Package 'python-psutil' has no installation candidate
```

## Create a new user account (loragw).

This `loragw` account will be used instead of default existing `pi` account for security reasons.
```shell
sudo useradd -m loragw -s /bin/bash
``` 

Type a suitable password for the `loragw` account.
```shell
sudo passwd loragw
``` 

Add the `loragw` user to the group `sudo` and allow sudo command with no password
```shell
sudo usermod -a -G sudo loragw
sudo cp /etc/sudoers.d/010_pi-nopasswd /etc/sudoers.d/010_loragw-nopasswd
sudo sed -i -- 's/pi/loragw/g' /etc/sudoers.d/010_loragw-nopasswd
``` 

copy default `pi` profile to `loragw`
```shell
sudo cp /home/pi/.profile /home/loragw/
sudo cp /home/pi/.bashrc /home/loragw/
sudo chown loragw:loragw /home/loragw/.*
``` 

Give `loragw` access to hardware I2C, SPI and GPIO
```shell
sudo usermod -a -G i2c,spi,gpio loragw
``` 

Now do some system configuration with `raspi-config` tool (bold are mandatory)
```shell
sudo raspi-config
``` 

  - network options, change hostname (loragw for example)
  - localization options, change keyboard layout
  - localization options, change time zone
  - **interfacing options, enable SPI, I2C** Serial and SSH (if not already done)
  - advanced options, expand filesystem
  - advanced options, reduce video memory split set to 16M

then do not when asked.

## Optionnal, Install log2ram this will preserve your SD card
```shell
git clone https://github.com/azlux/log2ram.git
cd log2ram
chmod +x install.sh uninstall.sh
sudo ./install.sh
sudo ln -s /usr/local/bin/ram2disk /etc/cron.hourly/
```


## Now reboot

**Reboot**
```shell
sudo reboot
```

## Reconnect after reboot

Log back with `loragw` user and if you changed hostname to loragw, use this command
```shell
ssh loragw@loragw.local
``` 


## Install nodejs

### For Pi Zero (see below for PI3)
``` 
sudo wget -O - https://raw.githubusercontent.com/sdesalas/node-pi-zero/master/install-node-v.lts.sh | bash
``` 

this will fire some error on unlink, but don't worry, that's ok because it's the first install.

Add the following to the end of your `~/.profile` file with `nano ~/.profile`
``` 
export PATH=$PATH:/opt/nodejs/bin
export NODE_PATH=/opt/nodejs/lib/node_modules
``` 

then **logout (CTRL-d) and log back** with `loragw` user* so new profile is loaded.
```shell
logout
``` 

### For Pi 3 (see above for PI Zero)
You can select and change nodejs version by changing for example `setup_8.x` to `setup_7.x`
``` 
sudo curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt-get install nodejs
``` 


## Install WS2812 driver for onboard LED (for RAK831 shield only)

The onboard WS2812 library and the Raspberry Pi audio both use the PWM, they cannot be used together. You will need to blacklist the Broadcom audio kernel module by editing a file 
``` 
echo "blacklist snd_bcm2835" | sudo tee --append /etc/modprobe.d/snd-blacklist.conf
``` 

### Install WS2812 led driver (for RAK831 shield only)
``` 
git clone https://github.com/jgarff/rpi_ws281x
cd rpi_ws281x/
scons
scons deb
sudo dpkg -i libws2811*.deb
sudo cp ws2811.h /usr/local/include/
sudo cp rpihw.h /usr/local/include/
sudo cp pwm.h /usr/local/include/
``` 

### Install WS2812 python wrapper (for RAK831 shield only)
``` 
cd python
python ./setup.py build
``` 

### Install WS2812 python library (for RAK831 shield only)
``` 
sudo python setup.py install
``` 

### Install NodeJS version of WS2812 driver (for RAK831 shield only)
``` 
cd
sudo npm install -g --unsafe-perm rpi-ws281x-native
npm link rpi-ws281x-native
``` 

## Install Python I2C/SPI OLED library (if you want to use OLED dispay)

Optionnaly you can add OLED to display usefull informations on it. Please look at this [documentation](https://github.com/ch2i/LoraGW-Setup/blob/master/doc/DisplayOled.md) for more informations

## Get CH2i Gateway Install repository
``` 
git clone https://github.com/ch2i/LoraGW-Setup
```

### Test WS2812 LED if you have any, in python or nodejs
Check that led color match the color displayed on console because on some WS2812B led, Red and Green could be reversed.
``` 
cd LoraGW-Setup
sudo ./testled.py
sudo ./testled.js
``` 

## Build and setup Packet Forwarder

New Multi-protocol Packet Forwarder by Jac @Kersing (thanks to @jpmeijers for scripting stuff)
Now build the whole thing, time to get a(nother) coffe, it can take 10/15 minutes!
``` 
sudo ./build.sh
``` 

## Build and setup legacy Packet Forwarder (optionnal)

If you really want to use the legacy packet forwarder you can launch this script. Both can be compiled on the same target, no problem, see below how to setup the legacy
``` 
sudo ./build_legacy.sh
``` 

## Configure Gateway on TTN console

Now you need to register your new GW on ttn before next step, see [gateway registration](https://www.thethingsnetwork.org/docs/gateways/registration.html#via-gateway-connector), fill the GW_ID and GW_KEY when running

## Launch TTN gateway configuration
This script will configure forwarder and all needed configuration 
``` 
sudo ./setup.sh
``` 

That's it, If you are using PI Zero [shield](https://github.com/hallard/RAK831-Zero), the 2 LED should be blinking green and you should be able to see your brand new gateway on TTN

# Usefull information

## Startup
Check all is fine also at startup, reboot your gateway.
``` 
sudo reboot
``` 

## LED Blinking colors (RAK831 Shied with 2 WS2812B Leds)

### LED 1

- green => connected to Internet
- blue  => No Internet connexion but gateway [WiFi AP](https://github.com/ch2i/LoraGW-Setup/blob/master/doc/AccessPoint.md) is up
- red => No Internet, no WiFi Access Point

### LED 2

- green => packet forwarder is started and running
- blue  => no packed forwarder but local [LoRaWAN server](https://github.com/ch2i/LoraGW-Setup/blob/master/doc/Lorawan-Server.md) is started
- red => No packet forwarder nor LoRaWAN server

## LED Blinking colors (iC880a with 4 GPIO Leds)

	- GPIO 4  (Blue) Blink => Internet access OK
	- GPIO 18 (Yellow) Blink => local web server up & running
	- GPIO 24 (Green)
		- Blink => packet forwarder is running
		- Fixed => Shutdown OK, can remove power
	- GPIO 23 (Red) 
		- Blink every second, one of the previous service down (local web, internet, )
	  - Middle bink on every bad LoRaWAN packet received
	  - Lot of short blink => Activity on SD Card (seen a boot for example)


### Change behaviour

You can change LED code behaviour at the end of script `/opt/loragw/monitor.py`


## Shutdown
You can press (and let it pressed) the switch push button, leds well become RED and after 2s start blinking in blue. If you release button when they blink blue, the Pi will initiate a shutdown. So let it 30s before removing power.

### Shutdown LED display (for iC880a only)

If you have a raspberry PI with this [IC880A shield](https://github.com/ch2i/iC880A-Raspberry-PI), and if you modded the `/boot/config.txt` file with following lines added into:

```
# When system if Halted/OFF Light Green LED
dtoverlay=gpio-poweroff,gpiopin=24
```
then the Green LED (gpio24) will stay on when you can remove the power of the gateway. It's really a great indicator.

You can also select which GPIO LED is used to replace activity LED if you need it.
```
# Activity LED
dtoverlay=pi3-act-led,gpio=23
```
then the Red LED (gpio23) will blink on activity.

## Detailled information

The installed sofware is located on `/opt/loragw`, I changed this name (original was ttn-gateway) just because not all my gateways are connected to TTN so I wanted to have a more generic setup.

```shell
ls -al /opt/loragw/
total 344
drwxr-xr-x 3 root root   4096 Jan 21 03:15 .
drwxr-xr-x 5 root root   4096 Jan 21 01:01 ..
drwxr-xr-x 9 root root   4096 Jan 21 01:03 dev
-rw-r--r-- 1 root root   6568 Jan 21 01:15 global_conf.json
-rwxr-xr-- 1 root root   3974 Jan 21 01:15 monitor-gpio.py
-rwxr-xr-- 1 root root   3508 Jan 21 03:15 monitor.py
-rwxr-xr-- 1 root root   4327 Jan 21 01:15 monitor-ws2812.py
-rwxr-xr-x 1 root root 307680 Jan 21 01:14 mp_pkt_fwd
-rwxr-xr-- 1 root root    642 Jan 21 01:36 start.sh
```

LED blinking and push button functions are done with the monitor.py service (launched by systemd at startup).
There are 2 versions of this service (with symlink), one with WS2812B led and another for classic GPIO LED such as the one on this [IC880A shield](https://github.com/ch2i/iC880A-Raspberry-PI). So if you want to change you can do it like that

### stop the service
```shell
sudo systemctl stop monitor
```

### If you have ic880a shield, change monitor service

In this case you do not have WS2812B RGB LED on the shield, but GPIO classic one. The push button GPIO to power off the PI is also not on the same GPIO, so you need to setup the correct monitor service.

```shell
sudo rm /opt/loragw/monitor.py
sudo ln -s /opt/loragw/monitor-gpio.py /opt/loragw/monitor.py
```

### start the service
```shell
sudo systemctl start monitor
```

### Check packed forwarder log 
```shell
sudo journalctl -f -u loragw
```


```
-- Logs begin at Sun 2018-01-21 14:57:08 CET. --
Jan 22 01:00:41 loragw loragw[240]: ### GPS IS DISABLED!
Jan 22 01:00:41 loragw loragw[240]: ### [PERFORMANCE] ###
Jan 22 01:00:41 loragw loragw[240]: # Upstream radio packet quality: 100.00%.
Jan 22 01:00:41 loragw loragw[240]: # Semtech status report send.
Jan 22 01:00:41 loragw loragw[240]: ##### END #####
Jan 22 01:00:41 loragw loragw[240]: 01:00:41  INFO: [TTN] bridge.eu.thethings.network RTT 52
Jan 22 01:00:41 loragw loragw[240]: 01:00:41  INFO: [TTN] send status success for bridge.eu.thethings.network
Jan 22 01:00:53 loragw loragw[240]: 01:00:53  INFO: Disabling GPS mode for concentrator's counter...
Jan 22 01:00:53 loragw loragw[240]: 01:00:53  INFO: host/sx1301 time offset=(1516578208s:159048µs) - drift=-55µs
Jan 22 01:00:53 loragw loragw[240]: 01:00:53  INFO: Enabling GPS mode for concentrator's counter.
Jan 22 01:01:11 loragw loragw[240]: ##### 2018-01-22 00:01:11 GMT #####
Jan 22 01:01:11 loragw loragw[240]: ### [UPSTREAM] ###
Jan 22 01:01:11 loragw loragw[240]: # RF packets received by concentrator: 0
Jan 22 01:01:11 loragw loragw[240]: # CRC_OK: 0.00%, CRC_FAIL: 0.00%, NO_CRC: 0.00%
Jan 22 01:01:11 loragw loragw[240]: # RF packets forwarded: 0 (0 bytes)
Jan 22 01:01:11 loragw loragw[240]: # PUSH_DATA datagrams sent: 0 (0 bytes)
Jan 22 01:01:11 loragw loragw[240]: # PUSH_DATA acknowledged: 0.00%
Jan 22 01:01:11 loragw loragw[240]: ### [DOWNSTREAM] ###
Jan 22 01:01:11 loragw loragw[240]: # PULL_DATA sent: 0 (0.00% acknowledged)
Jan 22 01:01:11 loragw loragw[240]: # PULL_RESP(onse) datagrams received: 0 (0 bytes)
Jan 22 01:01:11 loragw loragw[240]: # RF packets sent to concentrator: 0 (0 bytes)
Jan 22 01:01:11 loragw loragw[240]: # TX errors: 0
Jan 22 01:01:11 loragw loragw[240]: ### BEACON IS DISABLED!
Jan 22 01:01:11 loragw loragw[240]: ### [JIT] ###
Jan 22 01:01:11 loragw loragw[240]: # INFO: JIT queue contains 0 packets.
Jan 22 01:01:11 loragw loragw[240]: # INFO: JIT queue contains 0 beacons.
Jan 22 01:01:11 loragw loragw[240]: ### GPS IS DISABLED!
Jan 22 01:01:11 loragw loragw[240]: ### [PERFORMANCE] ###
Jan 22 01:01:11 loragw loragw[240]: # Upstream radio packet quality: 0.00%.
Jan 22 01:01:11 loragw loragw[240]: # Semtech status report send.
Jan 22 01:01:11 loragw loragw[240]: ##### END #####
Jan 22 01:01:11 loragw loragw[240]: 01:01:11  INFO: [TTN] bridge.eu.thethings.network RTT 53
Jan 22 01:01:11 loragw loragw[240]: 01:01:11  INFO: [TTN] send status success for bridge.eu.thethings.network
```


### Use legacy Packet Forwarder

If you want to use the legacy packet forwarder, you'll need to change file `/opt/loragw/start.sh` to replace the last line

```
./mp_pkt_fwd.sh
``` 
by
```
./poly_pkt_fwd.sh
``` 

```shell
sudo systemctl stop loragw
sudo systemctl start loragw
```


# And here is the final result

Click on image to see the video

[![CH2i RAK831 GW](http://img.youtube.com/vi/AZTomPGSOBY/0.jpg)](https://www.youtube.com/watch?v=AZTomPGSOBY "CH2i RAK831 GW")

# Add some other features

Here are other feature I use sometime on my gateways:

- Put the whole filesystem in [ReadOnly](https://hallard.me/raspberry-pi-read-only/)
- Setup PI as a WiFi [access point](https://github.com/ch2i/LoraGW-Setup/blob/master/doc/AccessPoint.md)
- Install a nice local [LoraWAN Server](https://github.com/ch2i/LoraGW-Setup/blob/master/doc/Lorawan-Server.md)
- Use a OLED [display](https://github.com/ch2i/LoraGW-Setup/blob/master/doc/DisplayOled.md) 
