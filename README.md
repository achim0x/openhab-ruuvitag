**Work in progress**

This repository is used to document setting up a raspberry pi with openHAB and collect sensordata from [ruuvitags](https://ruuvi.com/) based on pre-configured image:  
<https://f.ruuvi.com/t/collecting-ruuvitag-measurements-and-displaying-them-with-grafana/267/153>  
Download Link: <http://storage.ruuvi.com/ruuviberry_2018_04_18.img.zip>  
See also: <https://blog.ruuvi.com/setting-up-raspberry-pi-3-as-a-ruuvi-gateway-6e4a5b676510>

# Retention Policy

Connect to raspberry shell and type influx to start the influx db command line interface.

Switch to ruuvi database  
`> use ruuvi`

Change the duration of the default retention policy to e.g. 7 days. So that after 7 days the detailed data will be deleted.  
`> ALTER RETENTION POLICY autogen ON ruuvi DURATION 7d`

Create a new retention policy for the long term data  
`> CREATE RETENTION POLICY "forever" ON "ruuvi" DURATION INF REPLICATION 1`

Check the results  
`> SHOW RETENTION POLICIES`
```
name duration shardGroupDuration replicaN default
---- -------- ------------------ -------- -------
autogen 168h0m0s 168h0m0s 1 true
forever 0s 168h0m0s 1 false
```

Create a continuous query for the data that you would like to store long term.  
In this example the mean values of 30 minutes intervalls will be stored for temperature, humidity, pressure and battery voltage
```
> CREATE CONTINUOUS QUERY cq_30m ON ruuvi BEGIN
SELECT mean(temperature) as "mean_temperature", mean(humidity) as "mean_humidity",
mean(pressure) as "mean_pressure", mean(batteryVoltage) as "mean_batteryVoltage"
INTO ruuvi.forever.downsampled_measurements
FROM ruuvi.autogen.ruuvi_measurements GROUP BY time(30m), * FILL(null) END
```
# Grafana Dashboard mit Anonymem Zugriff

Auf der Console /etc/grafana/grafana.ini editieren
In dem Abschnitt Anonymous Auth folgendes eintragen:
```apache
#################################### Anonymous Auth ##########################
[auth.anonymous]
# enable anonymous access
enabled = true 
# specify organization name that should be used for unauthenticated users
org_name = Main Org.
# specify role for unauthenticated users
org_role = Viewer
```

`sudo systemctl restart grafana-server`

# Node Red
To publish ruuvitag measurements to openHab, mqtt is used. Collecting and publishing the measurements
is done by Node Red.  
Node Red also can write the measurements to the Influx DB, so the original Java based ruuvi collector
is not needed any more.
<https://lab.ruuvi.com/node-red/>
<https://tobru.ch/ruuvitag-with-c-h-i-p-node-red-influxdb-and-grafana/>

Auf dem Raspberry ist per default eine relativ alte Version von Node.js drauf. Diese sollte aktualisiert werden.
Siehe: Node Red on Raspberry Pi:
<https://nodered.org/docs/hardware/raspberrypi>  
```
bash <(curl -sL https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/update-nodejs-and-nodered)
```

Node Red als Service einrichten:
```
sudo systemctl enable nodered.service
sudo systemctl start nodered
```

Node Red ist ab sofort auf Port 1880 erreichbar

<https://flows.nodered.org/node/node-red-contrib-noble>
```shell
sudo apt-get install build-essential libudev-dev libbluetooth-dev

cd $HOME/.node-red
npm install node-red-contrib-noble
npm install node-red-contrib-influxdb

sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)
```

## Ruuvi Node installieren 
Verzeichnis anlegen, darin dann:
git clone https://github.com/achim0x/node-red
Im project root dann: sudo npm link
cd $HOME/.node-red
npm link node-red-contrib-ruuvitag

## Import Node
```
[{"id":"16282846.0cc62","type":"inject","z":"e53bb521.ce2ea8","name":"Start BLE scan every 30s","topic":"startBLEScan","payload":"{ \"scan\": true }","payloadType":"json","repeat":"30","crontab":"","once":true,"onceDelay":"","x":190,"y":160,"wires":[["38d7cf66.524e48"]]},{"id":"38d7cf66.524e48","type":"scan ble","z":"e53bb521.ce2ea8","uuids":"","duplicates":false,"name":"","x":560,"y":200,"wires":[["7d084d47.85e3dc","fb80d469.26252"]]},{"id":"7d084d47.85e3dc","type":"ruuvitag","z":"e53bb521.ce2ea8","name":"","x":730,"y":200,"wires":[["fb627207.a7988","8810dc6c.92c4f8","f1e7bb02.af96e8"]]},{"id":"96f777cb.31bfa","type":"inject","z":"e53bb521.ce2ea8","name":"Stop BLE scan every 30s","topic":"stopBLEScan","payload":"{ \"scan\": false }","payloadType":"json","repeat":"30","crontab":"","once":true,"onceDelay":"","x":190,"y":240,"wires":[["3cc34fd0.91721"]]},{"id":"76cb288f.9ab178","type":"debug","z":"e53bb521.ce2ea8","name":"","active":true,"console":"false","complete":"true","x":1130,"y":140,"wires":[]},{"id":"3cc34fd0.91721","type":"delay","z":"e53bb521.ce2ea8","name":"","pauseType":"delay","timeout":"10","timeoutUnits":"seconds","rate":"1","nbRateUnits":"1","rateUnits":"second","randomFirst":"1","randomLast":"5","randomUnits":"seconds","drop":false,"x":380,"y":240,"wires":[["38d7cf66.524e48"]]},{"id":"fb627207.a7988","type":"function","z":"e53bb521.ce2ea8","name":"Generate MQTT topic","func":"var device = null;\n       if (msg.peripheralUuid == \"f0977210bade\") {\n    device = \"wohnzimmer\";\n} else if (msg.peripheralUuid == \"eaa487776635\") { \n    device = \"bad\";\n} else if (msg.peripheralUuid == \"cb8cf2f41c53\") { \n    device = \"outdoor\";\n} else if (msg.peripheralUuid == \"dd4d8a54ba07\") { \n    device = \"dach\";\n} else if (msg.peripheralUuid == \"e6a2dfa0c9c8\") { \n    device = \"schlafzimmer\";\n} else if (msg.peripheralUuid == \"ffffffffffff\") {\n    device = \"bedroom3\";\n}\nmsg.topic = device + \"/measurements\";\nreturn msg;","outputs":1,"noerr":0,"x":940,"y":200,"wires":[["d655414c.0a9828","76cb288f.9ab178"]]},{"id":"d655414c.0a9828","type":"mqtt out","z":"e53bb521.ce2ea8","name":"","topic":"","qos":"","retain":"","broker":"6c05a863.74a028","x":1130,"y":200,"wires":[]},{"id":"8927add7.e545d8","type":"debug","z":"e53bb521.ce2ea8","name":"","active":false,"tosidebar":true,"console":false,"tostatus":false,"complete":"true","x":1130,"y":260,"wires":[],"inputLabels":["Input"]},{"id":"f1e7bb02.af96e8","type":"function","z":"e53bb521.ce2ea8","name":"Device naming","func":"var device = null;\nif (msg.peripheralUuid == \"f0977210bade\") {\n    device = \"Wohnzimmer\";\n} else if (msg.peripheralUuid == \"eaa487776635\") { \n    device = \"Bad\";\n} else if (msg.peripheralUuid == \"cb8cf2f41c53\") { \n    device = \"Outdoor\";\n} else if (msg.peripheralUuid == \"dd4d8a54ba07\") { \n    device = \"Dachzimmer\";\n} else if (msg.peripheralUuid == \"e6a2dfa0c9c8\") { \n    device = \"Schlafzimmer\";\n}\n\nvar newMsg = JSON.parse(msg.payload);\n\nvar formated = {\n    payload: [ newMsg, { tag: device, mac: msg.peripheralUuid} ]\n}\nreturn formated;","outputs":1,"noerr":0,"x":920,"y":320,"wires":[["8927add7.e545d8","5d13bcdc.f24574"]]},{"id":"5d13bcdc.f24574","type":"influxdb out","z":"e53bb521.ce2ea8","influxdb":"eb46006e.d8781","name":"","measurement":"ruuvi_measurements","precision":"","retentionPolicy":"","x":1200,"y":320,"wires":[]},{"id":"8810dc6c.92c4f8","type":"debug","z":"e53bb521.ce2ea8","name":"","active":false,"tosidebar":true,"console":false,"tostatus":false,"complete":"true","x":890,"y":140,"wires":[]},{"id":"fb80d469.26252","type":"debug","z":"e53bb521.ce2ea8","name":"","active":false,"tosidebar":true,"console":false,"tostatus":false,"complete":"true","x":750,"y":140,"wires":[]},{"id":"6c05a863.74a028","type":"mqtt-broker","z":"","name":"","broker":"localhost","port":"1883","clientid":"nodered","usetls":false,"compatmode":false,"keepalive":"60","cleansession":true,"willTopic":"","willQos":"0","willPayload":"","birthTopic":"","birthQos":"0","birthPayload":""},{"id":"eb46006e.d8781","type":"influxdb","z":"e53bb521.ce2ea8","hostname":"localhost","port":"8086","protocol":"http","database":"ruuvi","name":"influxdb","usetls":false,"tls":"e830c525.ef54b"},{"id":"e830c525.ef54b","type":"tls-config","z":"","name":"","cert":"","key":"","ca":"","certname":"","keyname":"","caname":"","verifyservercert":true}]
```

# MQTT Broker 
<https://wiki.fhem.de/wiki/MQTT_Einf%C3%BChrung>
<http://www.iot-airclean.at/datenaustausch-mit-mqtt/>

`sudo apt-get install mosquitto mosquitto-clients`
 
MQTT Server Test
```
sudo service mosquitto status
```

Start / Stop des Servers
```
 sudo service mosquitto stop
 sudo service mosquitto start
```

# OpenHab
## Update Java

<https://community.openhab.org/t/howto-install-zulu-embedded-java-on-raspberry-pi-3/22589>

```Bash
sudo su
apt-get install dirmngr
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 0x219BD9C9
echo 'deb http://repos.azulsystems.com/debian stable main' > /etc/apt/sources.list.d/zulu.list
apt-get update
apt-get install zulu-embedded-8
```

+ Oracle Java entfernen:  
   Achtung ein alternatives Java muss installiert werden. Auf jeden Fall bakckup davor machen.  
   `sudo apt-get remove --purge oracle-java8-jdk`

## Install OpenHAB
<https://docs.openhab.org/installation/linux.html#package-repository-installation>
```shell
wget -qO - 'https://bintray.com/user/downloadSubjectPublicKey?username=openhab' | sudo apt-key add -
sudo apt-get install apt-transport-https
echo 'deb https://dl.bintray.com/openhab/apt-repo2 stable main' | sudo tee /etc/apt/sources.list.d/openhab2.list
sudo apt-get update
sudo apt-get install openhab2
sudo apt-get install openhab2-addons
sudo systemctl start openhab2.service
sudo systemctl status openhab2.service
sudo systemctl daemon-reload
sudo systemctl enable openhab2.service
```

### JSON Path Transformation -
Das ist notwendig um die Sensorwerte aus dem JSON String zu extrahieren, der per MQTT empfangen wird
<https://github.com/openhab/openhab1-addons/wiki/Transformations>

+ Paper UI -> Add-ons -> Transformations
+ JSONPath Transformation installieren

### MQTT Binding -
<https://community.openhab.org/t/mqtt-binding-v1-11-getting-started-101/33958>

+ OpenHab Webinterface
+ PaperUI
+ Add-ons
+ Bindings
+ MQTT Binding installieren

#### Edit services/mqtt.cfg
```
mosquitto.url=tcp://127.0.0.1:1883
mosquitto ist hier die broker id, die dann auch in den items verwendet wird, und kann frei definiert werden
```

#### Create a items file per tag
```
String          ruuvi1Measurements       "Measurement:" { mqtt = "<[mosquitto:bad/measurements:state:default]" }
DateTime        ruuvi1LastUpdate         "Last Update [%1$td/%1$tm/%1$tY %1$tR]"    <calendar>
Number          ruuvi1Battery            "BatterieVoltage [%.2f V]"
Number          ruuvi1BatteryIcon        "Battery State [%d %%]"                    <battery>
Number          ruuvi1Humidity           "Humidity [%.2f %%]"              <humidity>
Number          ruuvi1Pressure           "Pressure [%.2f hPa]"             <pressure>
Number          ruuvi1Temperature        "Temperature [%.2f °C]"           <temperature>
Number          ruuvi1Rssi               "Rssi [%d] dBm"

```

#### Create a rules file per tag
<details><summary>Click to expand rules file</summary>
<p>

```
rule "ruuvi1Update"
when
        Item ruuvi1Measurements received update
then
        ruuvi1LastUpdate.postUpdate(new DateTimeType())

        var String json = (ruuvi1Measurements.state as StringType).toString

        var Double battery = new Double(transform("JSONPATH", "$.batteryVoltage", json))
        var Double humidity = new Double(transform("JSONPATH", "$.humidity", json))
        var Double pressure = new Double(transform("JSONPATH", "$.pressure", json)) / 100
        var Double temperature = new Double(transform("JSONPATH", "$.temperature", json))
        var Double rssi = new Double(transform("JSONPATH", "$.rssi", json))
        
        ruuvi1Battery.postUpdate(battery)
        if (                   battery < 2.50) {
                ruuvi1BatteryIcon.postUpdate(0)
        }
        if (battery >= 2.50 && battery < 2.55) {
                ruuvi1BatteryIcon.postUpdate(10)
        }
        if (battery >= 2.55 && battery < 2.60) {
                ruuvi1BatteryIcon.postUpdate(20)
        }
        if (battery >= 2.60 && battery < 2.65) {
                ruuvi1BatteryIcon.postUpdate(30)
        }
        if (battery >= 2.65 && battery < 2.70) {
                ruuvi1BatteryIcon.postUpdate(40)
        }
        if (battery >= 2.70 && battery < 2.75) {
                ruuvi1BatteryIcon.postUpdate(50)
        }
        if (battery >= 2.75 && battery < 2.80) {
                ruuvi1BatteryIcon.postUpdate(60)
        }
        if (battery >= 2.80 && battery < 2.85) {
                ruuvi1BatteryIcon.postUpdate(70)
        }
        if (battery >= 2.85 && battery < 2.90) {
                ruuvi1BatteryIcon.postUpdate(80)
        }
        if (battery >= 2.90 && battery < 2.95) {
                ruuvi1BatteryIcon.postUpdate(90)
        }
        if (battery >= 2.95) {
                ruuvi1BatteryIcon.postUpdate(100)
        }
        ruuvi1Humidity.postUpdate(humidity)
        ruuvi1Pressure.postUpdate(pressure)
        ruuvi1Temperature.postUpdate(temperature)
        ruuvi1Rssi.postUpdate(rssi)
end

```
<https://docs.openhab.org/v2.1/addons/iconsets/classic/readme.html>
</p>
</details>


#### Check eventlog if items are updated

`less /var/log/openhab2/eventlog`

#### Create Sitemap
<details><summary>Click to expand sitemap file</summary>
<p>

```
sitemap default label="My first sitemap"
{
  Frame label="Test Ruuvi Tag1"{
    Text item=ruuvi1Temperature
    Text item=ruuvi1Humidity
    Text item=ruuvi1Pressure
    Text item=ruuvi1Rssi
    Text item=ruuvi1LastUpdate
    Text item=ruuvi1BatteryIcon
    Text item=ruuvi1Battery icon="battery-0"    visibility=[ruuvi1BatteryIcon==0]
        Text item=ruuvi1Battery icon="battery10"        visibility=[ruuvi1BatteryIcon==10]
        Text item=ruuvi1Battery icon="battery20"        visibility=[ruuvi1BatteryIcon==20]
        Text item=ruuvi1Battery icon="battery30"        visibility=[ruuvi1BatteryIcon==30]
        Text item=ruuvi1Battery icon="battery40"        visibility=[ruuvi1BatteryIcon==40]
        Text item=ruuvi1Battery icon="battery50"        visibility=[ruuvi1BatteryIcon==50]
        Text item=ruuvi1Battery icon="battery60"        visibility=[ruuvi1BatteryIcon==60]
        Text item=ruuvi1Battery icon="battery70"        visibility=[ruuvi1BatteryIcon==70]
        Text item=ruuvi1Battery icon="battery80"        visibility=[ruuvi1BatteryIcon==80]
        Text item=ruuvi1Battery icon="battery90"        visibility=[ruuvi1BatteryIcon==90]
        Text item=ruuvi1Battery icon="battery100"       visibility=[ruuvi1BatteryIcon==100]
  }
}
```
</p>
</details>


Copy the battery level images battery10.png ... battery100.png to /etc/openhab2/icons/classic

Open PaperUI -> Configuration -> Services -> BasicUI  
Set Icon Format to bitmap

#### Setup Network Share
<https://docs.openhab.org/installation/linux.html#network-sharing>

# Usefull Influx DB commands 
## Influx DB Backup / Restore
<https://www.influxdata.com/blog/new-features-in-open-source-backup-and-restore/>
```shell
influxd backup -portable /tmp/backup_ex
influxd restore -portable /tmp/backup_ex
```

## Drop all measurements
`> drop series from downsampled_measurements`

# Todo for Sharing Image
+ Change pi user password
+ Change SMB openhab password
+ Clear commandline history
+ Delete DHCP cache
+ Shrink Image
