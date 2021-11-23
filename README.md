# RasPi OctoPrint in automated docker containers

Create OctoPrint docker containers that will automatically start/stop when a printer is connected/disconnected to the system

This is designed for the RaspberryPI 4b running the 64bit Bullseye OS
General steps include:
-install docker & docker-compose
-create first octoprint container and test connection to printer
-create service and enable
-create UDEV rules to start service
-restart UDEV service and test

## Docker & Docker-Compose installation
### Docker
Download and run the docker install script.
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo bash get-docker.sh
```
Add current user to the docker group to enable running without root
```
sudo usermod -aG docker $(whoami)
```

You must logout and back in to have changes take effect.

### Docker-Compose
First install python PIP:
```
sudo apt install python3-pip -y
```
Then install docker compose:
```
sudo pip3 install docker-compose
```

Now you can download octoprint container from the docker hub, https://hub.docker.com/r/octoprint/octoprint
```
docker pull octoprint/octoprint:latest
```

Edit the docker-compose.yml file and update for printer device settings and cameras test launch docker-compose.  Items to note are the paths to the printer and camera in the /dev directory. The current RasPI installs setup the 3d printers in the /dev/serial/by-id/ by default, at least for me. With no printers on or plugged in there is nothing listed in this directory.  Plug in the USB cable of the printer and turn it on, you should then see an entry in the direcotry:
```
pi@OctoPi:/dev/serial/by-id $ ls
usb-marlinfw.org_Marlin_USB_Device_0600800FAF291C895AA214DEF50020C7-if00
pi@OctoPi:/dev/serial/by-id $

```

The default yaml file has 2 octoprint services with their own profile (octo1, octo2) flag.  Usin the '--profile' flag on the command line will let us choose which one or both services will start.  When testing the docker-compose process, do not use '-d' with these as we want to see the output so we can correct any errors.
```
docker-compose --profile octo1 up
```
Make sure you can log into the octoprint service at the url, which should be port 80 of the IP address of the Pi.  If you can log in you can setup the OctoPrint service, all settings will be retained.
kill the session by stopping the docker-compose process
```
<ctrl>-c
```
You may now try to start the 2nd instance to test if needed and repeat.  The second container needs it's own external port if you want to be able to run both containers at the same time.  My default is set to 80 for the first container and 81 for the second.  I may update this in the future to use 8080 and 8081 to keep the default port 80 "empty".
``` 
docker-compose --profile octo2 up
```
Then kill this container once you can connect and verify the printer is working.
## Create the systemctl service for octoprint

## Create UDEV Rules
We also want to create a udev rule  to auto start/stop the octoprint service when the printer comes online.
Get the device idVender and idProduct values for creating the rules use udevadm.
```
udevadmn info -a /dev/serial/by-id/device_identifier 
```
Or you can use 'lusb' command and grep to get the information.
<pre>
$ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 005: ID 046d:0825 Logitech, Inc. Webcam C270
<b>Bus 001 Device 011: ID 1d50:6029 OpenMoko, Inc. Marlin 2.0 (Serial)</b>
Bus 001 Device 002: ID 2109:3431 VIA Labs, Inc. Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
</pre>
Then use lsusb using the -vs option and the 'Bus' and 'Device' id of the USB device and grep for the settings
```
 $ lsusb -vs 001:011 |grep "idVendor\|idProduct"
Couldn't open device, some information will be missing
  idVendor           0x1d50 OpenMoko, Inc.
  idProduct          0x6029 Marlin 2.0 (Serial)
```
You can ignore the part about not being able to open the device and the 0x in front of the ids indicates a hex code.  You will leave off the 0x part but use the 4 digit code after it in the udev rule.

You can see an example file in the udev directory of this repository, copy to /etc/udev/rules.d/ and edit, changing your device id's appropriatly.

activate new rules
```
sudo udevadm control --reload-rules
sudo systemctl restart systemd-udevd.service
```

Once done turn on or restart a printer and check that the octoprint service starts up.
```
sudo systemctl status octo1.service
```


