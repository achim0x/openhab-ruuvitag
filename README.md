This repository is used to document setting up a raspberry pi with openHAB and collect sensordata from [ruuvitags](https://ruuvi.com/) based on Raspbian Buster Lite:  
<https://www.raspberrypi.org/downloads/raspbian/>

See also: <https://blog.ruuvi.com/setting-up-raspberry-pi-3-as-a-ruuvi-gateway-6e4a5b676510>

After competing this guide the following services are running:
+ OpenHAB on port 8080
+ Influx DB on port 8086
+ Grafana on port 3000
+ Node Red on port 1880
+ Mosquitto on port 8083
+ Samba Network Share for openHAB config

# Setting up the Raspian
+ Download the Raspbian image
+ Under Windows use a tool like Win32DiskImager or Etcher to write the image to an SD Card
+ To activate SSH, go to the boot partition of the SD Card and create a file called "ssh" with no content
+ If you would like to setup WIFI or set a static IP Adress see here for more Information: <https://howtoraspberrypi.com/how-to-raspberry-pi-headless-setup/>
+ Connect the Raspi to the network and power it

After the first login via SSH the following steps shall be done
+ Update the installed Software
`sudo apt-get update`
and
`sudo apt-get upgrade`

+ Basic configuration with the tool raspi-config
`sudo raspi-config`
  + Change Password
  + Localisation Options
    + Change Time Zone
  + Advanced Options
    + Expand Filesystem to use complete SD card

Finish raspi-config and reboot to take over filesystem resizeing `sudo reboot`
  
For 24/7 operation it is a good idea to move the root fs to an external HD or SSD.
For Instructions see: https://www.carluccio.de/raspberry-pi-root-filesystem-auf-usb-festplatte/

# Installing Influx DB
add key for Influx repository
`curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -`
add repository
`echo "deb https://repos.influxdata.com/debian buster stable" | sudo tee /etc/apt/sources.list.d/influxdb.list`
`sudo apt-get update`
install
`sudo apt-get install -y influxdb`
start
`sudo systemctl start influxdb`
`sudo systemctl enable influxdb`


# Influx DB Retention Policy

When operating the ruuvi collector for longer time, a lot of data is generated when measurements are pushed into the database every few seconds.
For long term analysis a much rougher data grid is sufficient.  
Therfore in this example the original data will be stored for 7 days, and deleted after this time.
For long term storage 30 minutes mean values of some measurements will be stored.

Connect to raspberry shell and type influx to start the influx db command line interface.
`influx`

Create Databas
`CREATE DATABASE ruuvi`

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
> CREATE CONTINUOUS QUERY cq_30m ON ruuvi BEGIN SELECT mean(temperature) as "mean_temperature", mean(humidity) as "mean_humidity", mean(pressure) as "mean_pressure", mean(batteryVoltage) as "mean_batteryVoltage" INTO ruuvi.forever.downsampled_measurements FROM ruuvi.autogen.ruuvi_measurements GROUP BY time(30m), * FILL(null) END
```

# Installing Grafana
Check for latest Grafana release: https://grafana.com/grafana/download/6.5.3?platform=arm
and install
`sudo apt-get install -y adduser libfontconfig1`
`wget https://dl.grafana.com/oss/release/grafana-rpi_6.5.3_armhf.deb`
`sudo dpkg -i grafana-rpi_6.5.3_armhf.deb`
`sudo /bin/systemctl enable grafana-server`
`sudo /bin/systemctl start grafana-server`

everything should be running now. You can connect to Grafana on port 3000. Default user is admin/admin


# Grafana Dashboard mit Anonymem Zugriff

Open shell and edit: `/etc/grafana/grafana.ini`  
Loock for the chapter __Anonymous Auth__ and edit as shown:
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

<https://lab.ruuvi.com/node-red/>

On a default raspberry installation a quite old version of node.js is installed. This needs to be updated.  
See also: Node Red on Raspberry Pi:
<https://nodered.org/docs/hardware/raspberrypi>  
```shell
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
```

## Install Node Red as Service
```shell
sudo systemctl enable nodered.service
sudo systemctl start nodered
```

Now Node Red is available on Port 1880

## Install Node Noble
For accessing Bluetooth, Node Noble is used. See also:  
<https://flows.nodered.org/node/node-red-contrib-noble>
```shell
sudo apt-get install -y build-essential libudev-dev libbluetooth-dev

cd $HOME/.node-red

npm install @abandonware/noble
npm install MatsA/node-red-contrib-noble

sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)
```

## Install Node Influx DB
Node Red can write the measurements to the Influx DB. 
See also: <https://tobru.ch/ruuvitag-with-c-h-i-p-node-red-influxdb-and-grafana/>  
```
npm install node-red-contrib-influxdb
```

## Install Ruuvi Node
```
cd $HOME/.node-red
sudo apt-get install -y git-core
npm install ojousima/node-red
```

## Import Node
For importing of node see:
<https://github.com/achim0x/node-red/blob/master/ruuvi-node_example.md>

# MQTT Broker 
<https://wiki.fhem.de/wiki/MQTT_Einf%C3%BChrung>
<http://www.iot-airclean.at/datenaustausch-mit-mqtt/>

`sudo apt-get install -y mosquitto mosquitto-clients`
 
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
As recomended by openHAB community, install zulu java. See also:  
<https://community.openhab.org/t/howto-install-zulu-embedded-java-on-raspberry-pi-3/22589>

```shell
sudo su
apt-get install dirmngr
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 0x219BD9C9
echo 'deb http://repos.azulsystems.com/debian stable main' > /etc/apt/sources.list.d/zulu.list
apt-get update
apt-get install zulu-embedded-8
```

## Install OpenHAB
<https://docs.openhab.org/installation/linux.html#package-repository-installation>
```shell
cd $HOME/
wget -qO - 'https://bintray.com/user/downloadSubjectPublicKey?username=openhab' | sudo apt-key add -
sudo apt-get install -y apt-transport-https
echo 'deb https://dl.bintray.com/openhab/apt-repo2 stable main' | sudo tee /etc/apt/sources.list.d/openhab2.list
sudo apt-get update
sudo apt-get install -y openhab2 openhab2-addons openhab2-addons-legacy
sudo systemctl start openhab2.service
sudo systemctl status openhab2.service
sudo systemctl daemon-reload
sudo systemctl enable openhab2.service
```

### JSON Path Transformation
This is necessary for extracting the sensor values from the JSON string received via MQTT  
See also: <https://github.com/openhab/openhab1-addons/wiki/Transformations>

+ Open Paper UI -> Add-ons -> Transformations
+ Install JSONPath Transformation

### MQTT Binding
See also: <https://community.openhab.org/t/mqtt-binding-v1-11-getting-started-101/33958>

+ Open Paper UI -> Add-ons -> Bindings
+ Install MQTT Binding

#### Edit services/mqtt.cfg
```
mosquitto.url=tcp://127.0.0.1:1883
mosquitto ist hier die broker id, die dann auch in den items verwendet wird, und kann frei definiert werden
```

#### Timezone Setting
By default OpenHAB is using UTC, to change this to local time:  
`sudo nano /etc/default/openhab2`  
and add: `EXTRA_JAVA_OPTS="-Duser.timezone=Europe/Berlin"`

#### Create a items file per ruuvitag
[Ruuvitag example .item file](items/ruuvi1.items)

To show state dependent icons for the Battery and RSSI items, there are some user default icons.  
Copy the images from this git rep <icons/classic/> to `/etc/openhab2/icons/classic`

+ Open PaperUI -> Configuration -> Services -> BasicUI  
+ Set Icon Format to bitmap

For availabel default icons see: <https://docs.openhab.org/v2.1/addons/iconsets/classic/readme.html>
See also: <https://docs.openhab.org/v2.1/addons/iconsets/classic/readme.html>

#### Create a rules file per ruuvitag
[Ruuvitag example .rules file](rules/ruuvi1.rules)

#### Check eventlog if items are updated

`less /var/log/openhab2/eventlog`

#### Create Sitemap
[Example .sitemap file](sitemap/default.sitemap) 

#### Setup Network Share
To easily access openHab config, it is an good idea to setup samba to share the config files.  
See also: <https://docs.openhab.org/installation/linux.html#network-sharing>
```shell
sudo apt-get install -y samba samba-common-bin
sudo nano /etc/samba/smb.conf
```
Replace the default config file by this:
```
[global]
workgroup = WORKGROUP
security = user
encrypt passwords = yes

[openHAB2-userdata]
  comment=openHAB2 userdata
  path=/var/lib/openhab2
  browseable=Yes
  writeable=Yes
  only guest=no
  public=no
  create mask=0777
  directory mask=0777

[openHAB2-conf]
  comment=openHAB2 site configuration
  path=/etc/openhab2
  browseable=Yes
  writeable=Yes
  only guest=no
  public=no
  create mask=0777
  directory mask=0777
```
Set a password for SMB user openhab and restart service.
```shell
sudo smbpasswd -a openhab
sudo systemctl restart smbd.service
```

# Usefull Influx DB commands 
## Influx DB Backup / Restore
<https://www.influxdata.com/blog/new-features-in-open-source-backup-and-restore/>
```shell
influxd backup -portable /tmp/backup_ex
influxd restore -portable /tmp/backup_ex
```

## Drop all measurements
`> drop series from downsampled_measurements`

# Safe data to mysql using node red
## Install Node Red Node
cd $HOME/.node-red
npm install node-red-node-mysql
## Install Maria DB
sudo apt-get -y install mariadb-server mariadb-client
sudo mysql_secure_installation
## Install Apache / Php



INSERT INTO `Sensor_BleMinute` (`ID`, `sensor`, `temperature`, `humidity`, `preassure`, `timestamp`) VALUES (NULL, 'garage', '22,22', '99', '9999', CURRENT_TIMESTAMP);

apt-get -y install phpmyadmin

# Todo for Sharing Image
+ Change pi user password
+ Change SMB openhab password
+ Clear commandline history
+ Delete DHCP cache
+ Shrink Image
+ Replace MAC Adresses by dummy values in node red
