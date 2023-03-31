Установка инфраструктуры данных IoT через Docker

Основано на https://github.com/iothon/docker-compose-mqtt-influxdb-grafana и https://lucassardois.medium.com/handling-iot-data-with-mqtt-telegraf-influxdb-and-grafana-5a431480217.

Этот docker-compose устанавливает и настраивает:
- [Eclipse Mosquitto] (https://mosquitto.org) - открытый MQTT-брокер для сбора данных через протокол MQTT
- [InfluxDB] (https://www.influxdata.com/) - платформа для хранения данных по времени для хранения в базе данных временных рядов
- [Telegraf] (https://www.influxdata.com/time-series-platform/telegraf/) - серверный агент с открытым исходным кодом для подключения Mosquitto и InfluxDB друг к другу
- [Grafana] (https://grafana.com/) - открытая платформа для обеспечения наблюдаемости для создания графиков и других вещей.

# Процесс установки
## Установите Docker

sudo apt install docker.io
sudo apt установить docker-compose

sudo usermod -aG docker iothon

## Клонируйте этот репозиторий

git clone https://github.com/Ko1hozer/docker-compose-mosquitto-telegraf-influxdb-grafana.git

## Запустите это

Чтобы загрузить, настроить и запустить все службы, выполните
cd docker-compose-mosquitto-telegraf-influxdb-grafana
sudo docker-compose up -d

Чтобы проверить работающие службы, запустите
sudo docker ps

Чтобы выключить все, запустите
sudo docker-compose down

## Протестируйте вашу настройку

Опубликуйте несколько сообщений в вашем Mosquitto, чтобы вы уже могли увидеть некоторые данные в Grafana: 
sudo docker container exec mosquitto mosquitto_pub -t 'sensor/' -m '{"humidity":21, "temperature":21, "battery_voltage_mv":3000}'

### Grafana
Откройте в вашем браузере: 
http://<your-server-ip>:3000

Имя пользователя и пароль - admin:admin. Вы должны увидеть график данных, которые вы вводили с помощью команды mosquitto_pub.

### InfluxDB
Вы можете проверить свою настройку InfluxDB здесь:
http://<your-server-ip>:8086
Имя пользователя и пароль - user:password1234

# Конфигурация
### Mosquitto
Mosquitto настроен на разрешение анонимных подключений и размещения сообщений
listener 1883
allow_anonymous true

### InfluxDB
Конфигурация полностью находится в docker-compose.yml. Обратите внимание на DOCKER_INFLUXDB_INIT_ADMIN_TOKEN - вы можете запустить тест с указанным токеном, но вам лучше перегенерировать его для вашей собственной безопасности. Этот же токен повторяется в нескольких других файлах конфигурации, вы должны обновить его там также. Я не нашел легкого способа автоматически его генерировать в docker. Измените его, прежде чем вы это сделаете. Вы были предупреждены. Также измените имя пользователя и пароль.
  influxdb:
    image: influxdb
    container_name: influxdb
    restart: always
    ports:
      - "8086:8086"
    networks:
      - iot
    volumes:
      - influxdb-data:/var/lib/influxdb2
      - influxdb-config:/etc/influxdb2
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=user
      - DOCKER_INFLUXDB_INIT_PASSWORD=password1234
      - DOCKER_INFLUXDB_INIT_ORG=some_org
      - DOCKER_INFLUXDB_INIT_BUCKET=some_data
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=4eYvsu8wZCJ6tKuE2sxvFHkvYFwSMVK0011hEEiojvejzpSaij86vYQomN_12au6eK-2MZ6Knr-Sax201y70w==

### Telegraf
Telegraf отвечает за передачу mqtt-сообщений в influxdb. Он настроен на прослушивание темы sensor/. Вы можете изменять эту конфигурацию по своему усмотрению, проверьте официальную документацию о том, как это сделать. Обратите внимание на токен InfluxDB, который вы должны обновить.
[[inputs.mqtt_consumer]]
  servers = ["tcp://mosquitto:1883"]
  topics = [
    "sensor/#"
  ]
  data_format = "json"

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "4eYvsu8wZCJ6tKuE2sxvFHkvYFwSMVK0011hEEiojvejzpSaij86vYQomN_12au6eK-2MZ6Knr-Sax201y70w=="
  organization = "some_org"
  bucket = "some_data"
  
  ### Источник данных Grafana
Начально в Grafana установлен источник данных по умолчанию, указывающий на установленный в этом же compose экземпляр InfluxDB. Файл конфигурации находится в grafana-provisioning/datasources/automatic.yml. Обратите внимание на токен InfluxDB, который вы должны обновить.
apiVersion: 1

datasources:
  - name: InfluxDB_v2_Flux
    type: influxdb
    access: proxy
    url: http://influxdb:8086
    jsonData:
      version: Flux
      organization: some_org
      defaultBucket: some_data
      tlsSkipVerify: true
    secureJsonData:
      token: 4eYvsu8wZCJ6tKuE2sxvFHkvYFwSMVK0011hEEiojvejzpSaij86vYQomN_12au6eK-2MZ6Knr-Sax201y70w==

### Grafana мониторинг
Для настройки наблюдения Grafana находится каталог grafana-provisioning/dashboards.
