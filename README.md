# Интеграция Managed Service for Kafka в Yandex.Cloud и ksqlDB

В этом примере мы покажем пример настройки интеграции Managed Service for Kafka в Yandex.Cloud и ksqlDB

Настройка пройдет в несколько этапов

## Создание Kafka кластера

Для настройки нам понадобится настроенный yc (как это сделать можно посмотреть [тут](https://cloud.yandex.ru/docs/cli/quickstart)
* Создаем кластер в тестовой конфигурации (из одного брокера)
  ```bash
  yc kafka cluster create --zone-ids ru-central1-a --brokers-count 1 kafka-ksql --network-id <your-network-id>
  ```
- Создаем служебный топик
  ```bash
  yc kafka topic create --cluster-name kafka-ksql --partitions 1 --replication-factor 1 --cleanup-policy delete --retention-ms=-1 --min-insync-replicas 1 _confluent-ksql-default__command_topic
  ```
- Создаем еще один служебный топик
  ```bash
  yc kafka topic create --cluster-name kafka-ksql --partitions 3 --replication-factor 1 default_ksql_processing_log
  ```
- И топик из туториала ksqlDB для наших данных
  ```bash
  yc kafka topic create --cluster-name kafka-ksql --partitions 3 --replication-factor 1 locations
  ```
- Создаем пользователя для подключения ksqlDB
  ```bash
  yc kafka --cluster-name kafka-ksql users create --password KsqlPassword \
  --permission topic=_confluent-ksql-default__command_topic,role=ACCESS_ROLE_CONSUMER,role=ACCESS_ROLE_PRODUCER \
  --permission topic=default_ksql_processing_log,role=ACCESS_ROLE_CONSUMER,role=ACCESS_ROLE_PRODUCER \
  --permission topic=locations,role=ACCESS_ROLE_CONSUMER,role=ACCESS_ROLE_PRODUCER \
  ksql
  ```

## Настройка ksqlDB сервера

- Создаем виртуальную машину Compute на базе Ubuntu
  - В той же сети
  - С публичным IP адресом
  - С указанным ssh ключом для подключения
- Когда машина создастся, смотрим какой у нее публичный IP и подключаемся к ней по ssh
- Устанавливаем jre
  ```bash
  apt install openjdk-11-jre-headless
  ```
- Устанавливаем пакет _confluent-community-2.13_ с [сайта вендора](https://www.confluent.io/download/).
- Следуя инструкции для подключения к кластеру для Java, создаем хранилище для сертификата
  ```bash
  mkdir -p /usr/local/share/ca-certificates/Yandex
  wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" -O /usr/local/share/ca-certificates/Yandex/YandexCA.crt
  keytool -keystore client.truststore.jks -alias CARoot -import -file /usr/local/share/ca-certificates/Yandex/YandexCA.crt
  cp client.truststore.jks /etc/ksqldb
  ```
- Добавляем в конфиг _/etc/ksqldb/ksql-server.properties_ настройки подключения к кластеру.  FQDN брокера можно посмотреть на вкладке Хосты кластера
  ```
  bootstrap.servers=<broker_fqdn>:9091
  sasl.mechanism=SCRAM-SHA-512
  security.protocol=SASL_SSL
  ssl.truststore.location=/etc/ksqldb/client.truststore.jks
  ssl.truststore.password=<keystore_password (с шага создания хранилища)>
  sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="ksql" password="KsqlPassword";
  ```
- Запускаем сервис ksqlDB
  ```bash
  systemctl start confluent-ksqldb.service
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
