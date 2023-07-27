# RasPi OctoPrint in automated docker containers

Create OctoPrint docker containers that will automatically start/stop when a printer is connected/disconnected to the system

This is designed for the RaspberryPI 4b running the 64bit Bullseye OS
General steps include:
-create UDEV rules to give the printer a named symbolic link
-install docker & docker-compose
-create first octoprint container and test connection to printer
-create service and enable
-create UDEV rules to start service
-restart UDEV service and test

## Create symbolic link in /dev for printer(s)
Get the device idVender and idProduct values for creating the rules use udevadm

Use `lusb` command and grep to get the information.
```
$ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 005: ID 046d:0825 Logitech, Inc. Webcam C270
<b>Bus 001 Device 011: ID 1d50:6029 OpenMoko, Inc. Marlin 2.0 (Serial)</b>
Bus 001 Device 002: ID 2109:3431 VIA Labs, Inc. Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

Then use `lsusb -vs` and use the 'Bus' and 'Device' id of the USB device and grep for the settings
```
 $ lsusb -vs 001:011 |grep "idVendor\|idProduct"
Couldn't open device, some information will be missing
  idVendor           0x1d50 OpenMoko, Inc.
  idProduct          0x6029 Marlin 2.0 (Serial)
```
You can ignore the part about not being able to open the device and the 0x in front of the ids indicates a hex code.  
You will leave off the 0x part but use the 4 digit code after it in the udev rule.
Copy the file udev/99-setPrinter.rules to /etc/udev/rules.d/ and update it so that the 'idVendor' and 'idProduct' numbers match your printer.  Also update the SYMLINK to the name you want.
```
SUBSYSTEM=="tty",ATTRS{idVendor}=="1d50",ATTRS{idProduct}=="6029",SYMLINK+="ttyAM8"
```
Activate the new rules
```
sudo udevadm control --reload-rules
sudo systemctl restart systemd-udevd.service
```
Turn on the printer or unplug/plugin the USB and verify you can see the printer in /dev/ with the symbolink you specified.
```
ls -l ttyAM*
lrwxrwxrwx 1 root root 7 Jul 26 11:19 ttyAM8 -> ttyACM0
```
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

Edit the docker-compose.yml file and update for printer device settings and cameras test launch docker-compose.  Items to note are the paths to the printer and camera in the 'devices' section.
For the printer you can use your alias you configured earlier and usually the camera will be /dev/video0 for the first camera.  You may comment out or delete the line if you do not have a camera setup.

The default yaml file has 2 octoprint services with their own profile (octo1, octo2) flag.  Using the '--profile' flag on the command line will let us choose which one or both services will start.  When testing the docker-compose process, do not use '-d' with these as we want to see the output so we can correct any errors.
```
docker-compose --profile octo1 up
```
Make sure you can log into the octoprint service at the url, which should be port 8080 of the IP address of the Pi.  If you can log in you can setup the OctoPrint service, all settings will be retained.
kill the session by stopping the docker-compose process
```
<ctrl>-c
```
You may now try to start the 2nd instance to test if needed and repeat.  The second container needs it's own external port if you want to be able to run both containers at the same time.  
My default is set to 8080 for the first container and 8081 for the second. 
``` 
docker-compose --profile octo2 up
```
Then kill this container once you can connect and verify the printer is working.




## Create the systemctl service for octoprint

Copy the 'octo1.service' and 'octo2.service' files (if only using one printer, copy just octo1) to /etc/systemd/system/.  
These are the files that will start the OctoPrint service.  You will need to modify the user and possibly the path to the Docker files in these files as well as the device id.
See the example below
```
[Unit]
Description=OctoPrint Compose Service
**Requires=docker.service dev-ttyAM8.device**
**After=docker.service dev-ttyAM8.device**
StopWhenUnneeded=true
**BindsTo=dev-ttyAM8.device**

[Service]
User=ryan
Group=docker
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ryan/OctoPrint/octoprintDocker
ExecStart=/usr/local/bin/docker-compose --profile octo1 up -d
ExecStop=/usr/local/bin/docker-compose --profile octo1 down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```
To enable the new service:
```
sudo systemctl enable octo1
```
Check the status of the service to make sure it is found:
```
sudo systemctl status octo1
● octo1.service - OctoPrint Compose Service
     Loaded: loaded (/etc/systemd/system/octo1.service; enabled; vendor preset: enabled)
     Active: inactive (dead)

```

Now try to start the service:
```
sudo systemctl start octo1
```
If there are no errors output the service should be running and you can check by re-running the status command above.

Stop the service with:
```
sudo systemctl stop octo1
```

You can either start the service manually, or have the service start up anytime the printer is connected to the RaspberryPI and turned on by doing the next steps.

## Create UDEV Rule to start service

We also want to create a udev rule to auto start/stop the octoprint service when the printer comes online.


You can see an example file in the udev directory of this repository, copy to /etc/udev/rules.d/ and edit, changing your device id's appropriatly.

Activate the new rules
```
sudo udevadm control --reload-rules
sudo systemctl restart systemd-udevd.service
```

Once done turn on or restart a printer and check that the octoprint service starts up.
```
sudo systemctl status octo1.service
```

## Final checks

Once everything is done and services are reloaded and restarted you can run the final checks.  Plugin or turn on your printer to the PI and run `sudo systemctl status octo1` to verify that the docker has started.
Then check our the url on port 8080 (or whatever port you specified in the yml file).
Finally turn off the printer or remove the USB cable and the docker container and service should terminate on their own.
