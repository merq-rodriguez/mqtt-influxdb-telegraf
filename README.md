# Configuration Stack Technology Stack: INFLUXDB + TELEGRAF + MQTT


<img src="https://cms.gabrieltanner.org/content/images/2020/08/Technology-Stack.jpg"/>

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


# Scripts MQTT

<img src="https://storage.googleapis.com/jawahar-tech/1560609620174.png"/>

* Install paho.mqtt for golang:
```
  go get github.com/eclipse/paho.mqtt.golang
``` 

* Run:
```
  go run mqtt.go
```
<br>
<br>

# Run telegraf 

```
  docker run -d -p 3000:3000 grafana/grafana
```

Go to http://localhost:3000

<img src="https://cms.gabrieltanner.org/content/images/2020/08/grafana-login.png"/>

A continuación, navegue a la página para agregar fuente de datos haciendo clic en ***Configuración*** `>` ***Fuentes de datos*** en el menú lateral.

<img src="https://cms.gabrieltanner.org/content/images/2020/08/grafana-datasource.png"/>

Continúe seleccionando InfluxDB en el menú desplegable de bases de datos de series de tiempo. Ahora debe configurar la fuente de datos completando la URL y los campos de la base de datos según la configuración que configuró anteriormente.

La URL es una combinación de la dirección IP y el puerto de su API InfluxDB y la base de datos es el nombre de la base de datos que estableció anteriormente (Sensores si siguió el artículo).

<img src="https://cms.gabrieltanner.org/content/images/2020/08/grafana-influxdb-datasource.png"/>

Después de configurar y guardar correctamente la fuente de datos, puede continuar creando un nuevo panel. Siga los siguientes pasos para eso:

Haga clic en Nuevo panel .
Haga clic en Agregar nuevo panel . Grafana crea un panel gráfico básico que debe configurarse con sus datos de InfluxDB.

<img src="https://cms.gabrieltanner.org/content/images/2020/08/grafana-create-dashboard.png" />

Ahora es el momento de configurar su panel utilizando los datos de su base de datos InfluxDB. Aquí debe seleccionar el clima como la medida que desea utilizar para el campo desde. Luego, debe seleccionar la temperatura como campo.

Para obtener más información sobre el selector de consultas, <a href="https://grafana.com/docs/grafana/latest/datasources/influxdb/">visite la documentación oficial</a>.

# Escribir un punto en una serie

```
curl -i -XPOST 'http://localhost:8086/write?db=sensors' \
  --data-binary 'weather,host=server01,region=us-west value=0.64 1434055562000000000'
```

weather,location=us temperature=23232000