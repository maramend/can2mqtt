# can2mqtt
A Linux or Windows service to forward CAN frames to MQTT messages

# HowTo
Note: This whole readme assumes the following environment:
- You are running Rasbian on a Raspberry Pi, however any other Linux system should work
- You are using a USBtin or PiCAN device to connect to the CAN Bus (others might work)

## Install can-utils
```
sudo apt-get install git
cd ~
git clone https://github.com/linux-can/can-utils.git
cd can-utils
make
```

## Install Dotnet Core
```
wget https://download.visualstudio.microsoft.com/download/pr/5a496e41-23da-4aaa-94a7-baa9ab619fc6/0d000727345f3f71858ee79367f6ec23/dotnet-runtime-5.0.4-linux-arm.tar.gz
sudo tar -xvf ./dotnet-runtime-5.0.4-linux-arm.tar.gz -C /opt/dotnet/
sudo ln -s /opt/dotnet/dotnet /usr/local/bin
```

## Setup the CAN Bus connection 

For either USBtin or PiCAN

### USBtin
Assuming your device has the ID ttyACM0:
```
sudo ./slcan_attach -f -s1 -b 11 -o /dev/ttyACM0
sudo ./slcand ttyACM0 slcan0
sudo ifconfig slcan0 up
```

### PiCAN
Copy
```
# interfaces(5) file used by ifup(8) and ifdown(8)

# Please note that this file is written to be used with dhcpcd
# For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

# CAN-Bus
auto can0
iface can0 can static
  bitrate 50000
```

to `/etc/network/interfaces`. Adapt bitrate to your heating.

## Start canlogserver
This is only required to test or if you like to take care of canlogserver on your own. You can also configure a daemon to do this automatically (see below).
```
~/can-utils/canlogserver slcan0
```

## Start can2mqtt: 
This is only required to test or if you like to take care of can2mqtt on your own. You can also configure a daemon to do this automatically (see below).

Minimum parameter: `./can2mqtt_core --Daemon:MqttServer="192.168.0.192"`

All parameter: `./can2mqtt_core --Daemon:Name="Can2MqttSE" --Daemon:CanServer="192.168.0.192" --Daemon:CanServerPort=28700 --Daemon:MqttServer="192.168.0.192" --Daemon:MqttClientId="Can2Mqtt" --Daemon:MqttTopic="Heating" --Daemon:MqttTranslator="StiebelEltron" --Daemon:CanForwardWrite=true --Daemon:CanForwardRead=false --Daemon:CanForwardResponse=true`

### Startup Parameters:

| Parameter                     | Description                                                                                                                                                                                                                                                               | Default Value | Required | Example                                   |
|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------|----------|-------------------------------------------|
| `--Daemon:Name`               | Define the name of your Daemon                                                                                                                                                                                                                                            | Can2Mqtt      | No       | `--Daemon:Name="Can2MqttSE"`              |
| `--Daemon:CanServer`          | This is the address where your canlogserver is running                                                                                                                                                                                                                    | 127.0.0.1     | No       | `--Daemon:CanServer="192.168.0.192"`      |
| `--Daemon:CanServerPort`      | This is the port of the canlogserver                                                                                                                                                                                                                                      | 28700         | No       | `--Daemon:CanServerPort=28700`            |
| `--Daemon:MqttServer`         | This is the address of the MQTT Broker.                                                                                                                                                                                                                                   |               | Yes      | `--Daemon:MqttServer="192.168.0.192"`     |
| `--Daemon:MqttClientId`       | This is the clientId of the MQTT Client. Choose any name you like                                                                                                                                                                                                         | Can2Mqtt      | No       | `--Daemon:MqttClientId="Can2Mqtt"`        |
| `--Daemon:MqttTopic`          | This is the MQTT Root Topic of all MQTT message                                                                                                                                                                                                                           | Can2Mqtt      | No       | `--Daemon:MqttTopic="Heating"`            |
| `--Daemon:MqttTranslator`     | For some CAN Bus Clients are translators to translate the CAN Messages into a readable value and publish them via MQTT including the correct topic. Leave empty to publish every CAN frame without any further handling. Implemented translators right now: StiebelEltron |               | No       | `--Daemon:MqttTranslator="StiebelEltron"` |
| `--Daemon:CanForwardWrite`    | Should CAN frames of type "Write" be forwarded to MQTT?                                                                                                                                                                                                                   | true          | No       | `--Daemon:CanForwardWrite=true`           |
| `--Daemon:CanForwardRead`     | Should CAN frames of type "Read" be forwarded to MQTT?                                                                                                                                                                                                                    | false         | No       | `--Daemon:CanForwardRead=false`           |
| `--Daemon:CanForwardResponse` | Should CAN frames of type "Response" be forwarded to MQTT?                                                                                                                                                                                                                | true          | No       | `--Daemon:CanForwardResponse=true`        |
| `--Daemon:CANInterface`       | Detect CAN interface for proper decoding of CAN frames. `"auto"` will select the first interface in the system which contains "can" in the interface name, unless a specifc device name is configured, e.g `"can0"` or `"slcan0"`.                                        | auto          | No       | `--Daemon:CANInterface=can0`              |

## Configure and Register Daemon for can2mqtt and canlogserver
Execute `sudo nano etc/systemd/system/canlogserver.service` and paste the following into the file. Replace the slcan0 in case your socket has a different name:
```
[Unit]
Description=canlogserver
After=network.target

[Service]
ExecStart=/home/pi/can-utils/canlogserver slcan0
WorkingDirectory=/home/pi/can-utils/canlogserver
StandardOutput=inherit
StandardError=inherit
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```
Run `sudo systemctl start canlogserver.service` to test if the service starts (it should). To see if it runs, execute `sudo systemctl status canlogserver.service`. You should see something similar like this if everything went well in the last line: `Jul 03 19:09:16 raspi-test systemd[1]: Started canlogserver.`

Now we repeat this for the can2mqtt daemon. Execute `sudo nano etc/systemd/system/can2mqtt.service` and paste the following into the file. Replace the placeholder with your parameters:
```
[Unit]
Description=can2mqtt
After=network.target

[Service]
ExecStart=/usr/local/bin/dotnet /home/pi/can2mqtt_core/can2mqtt_core.dll >>>STATE YOUR PARAMETERS HERE<<<
WorkingDirectory=/home/pi/can2mqtt_core/
StandardOutput=inherit
StandardError=inherit
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```
An example for the ExecStart line is:
```ExecStart=/usr/local/bin/dotnet /home/pi/can2mqtt_core/can2mqtt_core.dll --Daemon:MqttServer="127.0.0.1" --Daemon:MqttClientId="Can2Mqtt" --Daemon:CANInterface="slcan0"```

To test if it works, run the following command: `sudo systemctl start can2mqtt.service`. To test if the service is running, execute `sudo systemctl status can2mqtt.service`. You will see the last lines of output. If you do not see any errors after about 10 seconds, everything is fine.

Finally if everything is fine and you want to autostart the canlogserver and can2mqtt application, execute this: 
```
sudo systemctl enable canlogserver.service
sudo systemctl enable can2mqtt.service
```