# fiat_vehicle_mqtt
Access Fiat vehicle data and send it periodically to an MQTT broker. Send command requests (e.g. lock/unlock the door) to your car.

Fork from https://github.com/mahil4711/fiat_vehicle_mqtt including my changes:
1) Added 2 datapoints: **batteryVoltage** (... but I dont know which Battery... value did not update...?? Added anyway) and **chargePowerPreference** (This is the charging speed set by vehicle, Level 1 to Level 5)
2) Logging added to a MQTT item. The log message is sent to MQTT, item: fiat/"your vin"/lastLog . So this should hold the last issued Log Message.
3) Set a timezone for logging timestamps. The timezone may be configured in config file **fiat.cfg** Timezone section. Valid timezone strings can be found here: https://www.php.net/manual/en/timezones.php
4) Possibility to disable periodic data fetch from fiat cloud. If item **sleep** or **sleep_charging** in fiat.cfg is left empty then no periodic data read from fiat is happening. Data is read just one time at script start. A timestamp is sent every 20 seconds to MQTT item: fiat/php_time . So user has to trigger a data read manually by sending string "UPDATE" to the command MQTT topic fiat/"your vin"/command . Note: If the last data request is older than 5 minutes and car is charging a DEEPREFRESH is issued automatically (see function apiRequestALL in api.php)


## Sources
This software is based on [api.php](https://github.com/schmidmuc/fiat_vehicle/blob/main/api.php) from https://github.com/schmidmuc/fiat_vehicle with some small adjustments.

## Prerequisites
This software needs PHP 7.x to work and a running MQTT broker.

## Configuration
After downloading the software you just need to create the configfile __fiat.cfg__ with at least the settings for your MQTT broker and your Uconnect account in the software download directory:
```
[mqtt]
server = <IP of your MQTT broker>
port = <port of your MQTT broker>
username = <username for MQTT broker, leave empty if not needed>
password = <password for MQTT broker, leave empty if not needed>

[fiat]
username = <Uconnect account email>
password = <Uconnect account password>
PIN = <PIN used in the Fiat app>
sleep = 300
sleep_charging = 60
GoogleApiKey = "<optional GoolgeApi key>"
```

The __sleep__ parameter defines the wait time between rereading the Fiat vehicle data. The above example means, that the data will be updated every 300 seconds by default. If the car is charing the update time will be set to 60 seconds according to the parameter __sleep_charging__.

The [GoogleApiKey](https://support.google.com/googleapi/answer/6158862?hl=en) is needed to translate the latitude/longitude data read from your car to a real address. You could leave this empty if you do not need this.

## Using docker to run this software
Goto the download directory of this software. Adjust the TZ variable in __Dockerfile__ to your needs and run the following commands:
- docker build -t fiat .
- docker-compose up -d

This will configure and start a docker container named __fiat__. 

## Running this software from the command line
The Debian packages __php-xml php-curl composer__ (or similar ones from other distributions) needs to be installed. Afterwards run __composer install__ in the directory where you have downloaded this software. Now run __php fiat_mqtt.php__.

## Sending commands to your car
The following commands are implemented:
```
  $commands = array (
    "VF"            => "location",     // UpdateLocation (updates gps location of the car)
    "DEEPREFRESH"   => "ev",           // DeepRefresh (same as "RefreshBatteryStatus")
    "HBLF"          => "remote",       // Blink (blink lights)
    "CNOW"          => "ev/chargenow", // ChargeNOW (starts charging)
    "ROTRUNKUNLOCK" => "remote",       // Unlock trunk
    "ROTRUNKLOCK"   => "remote",       // Lock trunk
    "RDU"           => "remote",       // Unlock doors
    "RDL"           => "remote",       // Lock doors
    "ROPRECOND"     => "remote",       // Turn on/off HVAC
  );

```
To e.g. lock the doors you could use the command:
```
mosquitto_pub -h <IP of MQTT broker> -p <port of MQTT broker> -m "RDL" -t fiat/<VIN of your car>/command
```
Maybe not all commands are working on each car. This is the list of commands I found in [api.php](https://github.com/schmidmuc/fiat_vehicle/blob/main/api.php).

Additionally the command __UPDATE__ was added to immediately reread the Fiat vehicle data.

## Status
- In general it should be possible to get data for multiple vehicles attached to a given account. The MQTT client key is based on the VIN. So each car generates a differnet MQTT client key. As I only have one car, I'm not able to test this.
- Tested with a Fiat 500e 2021.
- There is a lot of other data available via [api.php](https://github.com/schmidmuc/fiat_vehicle/blob/main/api.php). Only a subset is written to the MQTT broker. To see which data is currently published to the MQTT broker, please check the variable $payload in fiat_get_data() in [this](https://github.com/mahil4711/fiat_vehicle_mqtt/blob/main/fiat_mqtt.php) file.
