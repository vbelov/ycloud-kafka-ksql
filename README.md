# Интеграция Managed Service for Kafka в Yandex.Cloud и ksqlDB

В этом примере мы покажем пример настройки интеграции Managed Service for Kafka в Yandex.Cloud и ksqlDB

Настройка пройдет в несколько этапов

## Создание Kafka кластера

Для настройки нам понадобится настроенный yc (как это сделать можно посмотреть [тут](https://cloud.yandex.ru/docs/cli/quickstart)
* Создаем кластер в тестовой конфигурации (из одного брокера)
  ```bash
  yc kafka cluster create --version 3.4 --zone-ids ru-central1-a --brokers-count 1 --network-name default --log-segment-bytes 104857600 kafka-ksql
  ```
- Создаем топик из туториала ksqlDB для наших данных
  ```bash
  yc kafka topic create --cluster-name kafka-ksql --partitions 3 --replication-factor 1 locations
  ```
- Создаем пользователя для подключения ksqlDB
  ```bash
  yc kafka --cluster-name kafka-ksql users create --password KsqlPassword --permission topic=*,role=admin ksql
  ```

## Настройка ksqlDB сервера

- Создаем виртуальную машину Compute на базе Ubuntu:
  ```bash
  yc compute instance create \ 
  --create-boot-disk type=network-hdd,size=40,image-family=ubuntu-2004-lts,image-folder-id=standard-images \
  --network-interface subnet-id=<YOUR_SUBNET_ID>,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/id_rsa.pub \
  --zone ru-central1-a \
  --name ksql \
  --cores 4 --memory 16G
  ```
  Можно попробовать создать инстанс с меньшим количеством ресурсов.
- Когда машина создастся, смотрим какой у нее публичный IP и подключаемся к ней по ssh
- Устанавливаем jre
  ```bash
  apt install openjdk-11-jre-headless
  ```
- Скачиваем и устанавливаем _confluent-community_ с [сайта вендора](https://www.confluent.io/previous-versions/). Туториал проверен на версии 7.4.1.
- Следуя инструкции для подключения к кластеру для Java, создаем хранилище для сертификата (команда keytool запросит пароль, которым будет защищен доступ к хранилищу, в рамках данного туториала укажите в качестве пароля `keystore_password`)
  ```bash
  sudo mkdir -p /usr/local/share/ca-certificates/Yandex
  sudo wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" -O /usr/local/share/ca-certificates/Yandex/YandexCA.crt
  keytool -keystore client.truststore.jks -alias CARoot -import -file /usr/local/share/ca-certificates/Yandex/YandexCA.crt
  ```
- Добавляем в конфиг _./confluent-7.4.1/etc/ksqldb/ksql-server.properties_ настройки подключения к кластеру.  FQDN брокера можно посмотреть на вкладке Хосты кластера
  ```
  bootstrap.servers=<broker_fqdn>:9091
  sasl.mechanism=SCRAM-SHA-512
  security.protocol=SASL_SSL
  ssl.truststore.location=/home/yc-user/client.truststore.jks
  ssl.truststore.password=keystore_password
  sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="ksql" password="KsqlPassword";
  ```
- Запускаем сервис ksqlDB
  ```bash
  ./confluent-7.4.1/bin/ksql-server-start ./confluent-7.4.1/etc/ksqldb/ksql-server.properties
  ```

## Пример использования

Взято из [официального туториала](https://ksqldb.io/quickstart.html)
- Запускаем клиент ksql (с виртуальной машины ksqlDB сервера) и в нем создаем подключение к существующему топику
  ```sql
  CREATE STREAM riderLocations (profileId VARCHAR, latitude DOUBLE, longitude DOUBLE) WITH (kafka_topic='locations', value_format='json', partitions=3);
  ```
- И создаем запрос к данным
  ```sql
  SELECT * FROM riderLocations WHERE GEO_DISTANCE(latitude, longitude, 37.4133, -122.1162) <= 5 EMIT CHANGES;
  ```
- Предыдущий запрос показывает данные по мере появления, поэтому мы его не закрываем, а открываем еще одну вкладку терминала и в ней запускаем еще один ksql. В нем мы будем добавлять тестовые данные
  ```sql
  INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('c2309eec', 37.7877, -122.4205);
  INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('18f4ea86', 37.3903, -122.0643);
  INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('4ab5cbad', 37.3952, -122.0813);
  INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('8b6eae59', 37.3944, -122.0813);
  INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('4a7c7b41', 37.4049, -122.0822);
  INSERT INTO riderLocations (profileId, latitude, longitude) VALUES ('4ddad000', 37.7857, -122.4011);
  ```
- И в первой вкладке видим появившиеся строчки, удовлетворившие условию
  ```
  +--------------------------+--------------------------+---------------------------+
  |PROFILEID                 |LATITUDE                  |LONGITUDE                  |
  +--------------------------+--------------------------+---------------------------+
  |4ab5cbad                  |37.3952                   |-122.0813                  |
  |8b6eae59                  |37.3944                   |-122.0813                  |
  |4a7c7b41                  |37.4049                   |-122.0822                  |
  ```

## Настройка логирования сервера ksql в Kafka-топик

Создаем топик для логов:
```bash
yc kafka topic create --cluster-name kafka-ksql --partitions 3 --replication-factor 1 KSQL_LOG
```

В файл `./confluent-7.4.1/etc/ksqldb/log4j.properties` добавляем следующие строки:
```
log4j.appender.kafka_appender=org.apache.kafka.log4jappender.KafkaLog4jAppender
log4j.appender.kafka_appender.layout=io.confluent.common.logging.log4j.StructuredJsonLayout
log4j.appender.kafka_appender.BrokerList=<broker_fqdn>:9091
log4j.appender.kafka_appender.Topic=KSQL_LOG
log4j.logger.io.confluent.ksql=INFO,kafka_appender

log4j.appender.kafka_appender.clientJaasConf=org.apache.kafka.common.security.scram.ScramLoginModule required username="ksql" password="KsqlPassword";
log4j.appender.kafka_appender.SecurityProtocol=SASL_SSL
log4j.appender.kafka_appender.SaslMechanism=SCRAM-SHA-512
log4j.appender.kafka_appender.SslTruststoreLocation=/home/yc-user/client.truststore.jks
log4j.appender.kafka_appender.SslTruststorePassword=keystore_password
```

Перезапускаем ksql сервер:
```bash
export KSQL_LOG4J_OPTS="-Dlog4j.configuration=file:/home/yc-user/confluent-7.4.1/etc/ksqldb/log4j.properties"
./confluent-7.4.1/bin/ksql-server-start ./confluent-7.4.1/etc/ksqldb/ksql-server.properties
```

Устанавливаем kafkacat и начинаем читать лог:
```bash
sudo apt install kafkacat

kafkacat -b <broker_fqdn>:9091\
 -t KSQL_LOG \
 -X security.protocol=SASL_SSL \
 -X sasl.mechanisms=SCRAM-SHA-512 \
 -X sasl.username=ksql \
 -X sasl.password=KsqlPassword \
 -X ssl.ca.location=/usr/local/share/ca-certificates/Yandex/YandexCA.crt \
 -Z -K:
```
