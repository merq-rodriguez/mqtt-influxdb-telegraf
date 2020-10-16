# Configuration MQTT + TELEGRAF + MQTT

# In the host:

* Download telegraf in host:

```
  wget https://dl.influxdata.com/telegraf/releases/telegraf_1.14.0-1_amd64.deb
```
* Install file `.deb`

```
  sudo dpkg -i telegraf_1.14.0-1_amd64.deb
```
Add user telegraf to group docker:


```
  sudo usermod -aG docker telegraf
```


* Restart relegraf:


```
  systemctl restart telegraf
```

<br>
<br>
<br>

# With Docker:
### 1. Create network with name `influxdb`

```
  docker network create influxdb
```

#### 2. Run influxDB with docker


```
docker run -d -p 8086:8086 --net=influxdb -v influxdb:/var/lib/influxdb --name influxdb influxdb
```


<br>
<br>
<br>

# Configuration

* Enter to container influxdb:
```
  docker exec -it influxdb influx
```
* Create database `sensors` and user `telegraf`

```
  CREATE DATABASE sensors
  CREATE USER telegraf WITH PASSWORD 'telegraf'
  GRANT ALL ON sensors TO telegraf
```
 
* Later we will continue to edit sample file `telegraf.conf` in the host:

```
sudo nano /etc/telegraf/telegraf.conf
```

* Or if it using docker:

First, generate a sample configuration and save it as  `telegraf.conf` on the host:

```docker
  docker run --rm telegraf telegraf config > telegraf.conf
```

Here we are going to modify, only the following sections, the ones that are from "Output Plugins", look for them in the file.

The first thing we are going to specify is how it will connect to InfluxDB, don't forget that this is going to be the database that will store the metrics that Telegraf "spits out".

```conf
[[outputs.influxdb]]
  urls = ["http://influxdb:8086"]
  database = "telegraf"
  skip_database_creation = true
  username = "telegraf"
  password = "telegraf"
```

Use `skip_database_creation = true` if already exists the database, else set `skip_database_creation = false`.

This is the first thing we are going to modify and of course, we are going to uncomment. So that it is able to read the Docker socket. By default `[[inputs.docker]]` appears commented, so we are going to uncomment it as well, as it appears below.

```conf
[[inputs.docker]]
  endpoint = "unix:///var/run/docker.sock"
```

Here you will need to modify some fields under the `[[inputs.mqtt_consumer]]` tag, which defines your MQTT broker information.

```conf
[[inputs.mqtt_consumer]]
  servers = ["tcp://mqtt:1883"]
  topics = [
    "sensors"
  ]
  data_format = "influx"
```




### Run docker container
* In important set `--net=influxdb` and the file of configuration `telegraf.conf`

```docker
  docker run -d --name=telegraf \
      --net=influxdb \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v $PWD/telegraf.conf:/etc/telegraf/telegraf.conf:ro \
      telegraf
```

# Run telegraf 

```
  docker run -d -p 3000:3000 grafana/grafana
```

