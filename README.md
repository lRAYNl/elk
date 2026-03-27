# ELK Stack — Elasticsearch + Logstash + Kibana

Развёртывание трёхузлового кластера **Elasticsearch 9.2.2** с Logstash и Kibana.  
Транспорт и HTTP защищены TLS. Логи поступают из Kafka через Logstash.

```
git clone https://github.com/lRAYNl/elk.git ~/elk
```

---

## Содержание

- [Архитектура](#архитектура)
- [Требования](#требования)
- [Структура репозитория](#структура-репозитория)
- [Шаг 1 — Генерация SSL-сертификатов ELK](#шаг-1--генерация-ssl-сертификатов-elk)
- [Шаг 2 — Добавление Kafka CA в Logstash truststore](#шаг-2--добавление-kafka-ca-в-logstash-truststore)
- [Шаг 3 — Распределение сертификатов](#шаг-3--распределение-сертификатов)
- [Шаг 4 — Настройка .env](#шаг-4--настройка-env)
- [Шаг 5 — Запуск кластера](#шаг-5--запуск-кластера)
- [Шаг 6 — Установка пароля kibana\_system](#шаг-6--установка-пароля-kibana_system)
- [Шаг 7 — Настройка ILM](#шаг-7--настройка-ilm-index-lifecycle-management)
- [Шаг 8 — Настройка S3 snapshot-репозитория](#шаг-8--настройка-s3-snapshot-репозитория)
- [Шаг 9 — Проверка системы](#шаг-9--проверка-системы)
- [Изменение конфигурации Logstash pipeline](#изменение-конфигурации-logstash-pipeline)
- [Полезные команды](#полезные-команды)
- [Устранение неполадок](#устранение-неполадок)

---

## Архитектура

```
[Kafka кластер]  ──→  Logstash  ──→  Elasticsearch (HTTPS :9200)
                                           │
                                       Kibana (HTTPS :5601)
```

Каждый ELK-узел запускает три контейнера: `elasticsearch`, `logstash`, `kibana`.

| Сервис        | Порт       | Протокол |
|---------------|------------|----------|
| Elasticsearch | 9200, 9300 | HTTPS    |
| Kibana        | 5601       | HTTPS    |
| Logstash      | —          | internal |

---

## Требования

- Docker 24.0+
- Docker Compose 2.20+
- OpenSSL 3.0+
- ОЗУ: минимум 4 ГБ на узел (Elasticsearch занимает 2 ГБ heap)

### Установка Docker и обязательная системная настройка

Выполнить на **каждом** ELK-узле:

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt install docker-compose-v2 -y

# Обязательно — без этого Elasticsearch не запустится
sudo sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
```

---

## Структура репозитория

```
elk/
├── docker-compose.yml
├── .env.template
├── logstash.conf          # переопределение pipeline (опционально)
└── certs/                 # создаётся вручную (см. Шаг 1)
    ├── ca.crt
    ├── ca.key             # только на elk-node-01!
    ├── ca.srl
    ├── elastic-certificates.p12
    ├── elastic-stack-ca.p12   # только на elk-node-01!
    ├── kibana.crt
    ├── kibana.key
    ├── logstash.crt
    ├── logstash.key
    ├── logstash-truststore.p12  # содержит оба CA: ELK-CA + Kafka-CA
    ├── kafka-ca.crt         # CA от Kafka-кластера
    ├── kafka.crt
    └── kafka.key
```

---

## Шаг 1 — Генерация SSL-сертификатов ELK

> Все команды выполняются на **elk-node-01** в директории `~/elk/certs/`.

```bash
mkdir -p ~/elk/certs && cd ~/elk/certs
```

### 1.1 Корневой CA для ELK

```bash
openssl genrsa -out ca.key 4096

openssl req -new -x509 -days 3650 \
  -key ca.key \
  -out ca.crt \
  -subj "/CN=ELK-CA/O=MyOrg/C=RU"
```

### 1.2 Сертификаты Elasticsearch (формат PKCS12)

```bash
# Вернуться в директорию elk (важно для --user флага)
cd ~/elk

# Создать CA в формате p12
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$(pwd)/certs:/certs" \
  docker.elastic.co/elasticsearch/elasticsearch:9.2.2 \
  bin/elasticsearch-certutil ca \
    --out /certs/elastic-stack-ca.p12 \
    --pass ""

# Создать сертификат узлов ES
docker run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$(pwd)/certs:/certs" \
  docker.elastic.co/elasticsearch/elasticsearch:9.2.2 \
  bin/elasticsearch-certutil cert \
    --ca /certs/elastic-stack-ca.p12 \
    --ca-pass "" \
    --out /certs/elastic-certificates.p12 \
    --pass ""

cd certs/
```

### 1.3 Сертификат Kibana

```bash
openssl genrsa -out kibana.key 2048

openssl req -new -key kibana.key -out kibana.csr \
  -subj "/CN=kibana/O=MyOrg/C=RU"

openssl x509 -req -days 3650 \
  -in kibana.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out kibana.crt
```

### 1.4 Сертификат Logstash

```bash
openssl genrsa -out logstash.key 2048

openssl req -new -key logstash.key -out logstash.csr \
  -subj "/CN=logstash/O=MyOrg/C=RU"

openssl x509 -req -days 3650 \
  -in logstash.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out logstash.crt
```

### 1.5 Создать начальный Logstash truststore (из ELK CA)

```bash
openssl pkcs12 -export \
  -nokeys \
  -in ca.crt \
  -out logstash-truststore.p12 \
  -passout pass:changeit
```

### 1.6 Сертификат Kafka (для Logstash-клиента, опционально)

```bash
openssl genrsa -out kafka.key 2048

openssl req -new -key kafka.key -out kafka.csr \
  -subj "/CN=kafka/O=MyOrg/C=RU"

openssl x509 -req -days 3650 \
  -in kafka.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out kafka.crt
```

---

## Шаг 2 — Добавление Kafka CA в Logstash truststore

> **Это обязательный шаг.** Kafka и ELK используют **разные CA** (Kafka-CA и ELK-CA).  
> Logstash должен доверять обоим, иначе получит ошибку `Failed to create new NetworkClient`.

### 2.1 Получить ca.crt от Kafka-кластера

Запустить **на kafka-node-01** HTTP-сервер для раздачи файла:

```bash
cd ~/kafka/certs
python3 -m http.server 8888
```

Скачать **на elk-node-01**:

```bash
cd ~/elk/certs
wget http://<IP_KAFKA_NODE_01>:8888/ca.crt -O kafka-ca.crt
```

Остановить сервер на kafka-node-01 (`Ctrl+C`).

Проверить что CA действительно разные:

```bash
openssl x509 -in ca.crt -noout -subject        # должно быть ELK-CA
openssl x509 -in kafka-ca.crt -noout -subject  # должно быть Kafka-CA
```

### 2.2 Добавить Kafka CA в существующий truststore

Используем keytool из образа Logstash (чтобы не зависеть от системной Java):

```bash
cd ~/elk

sudo docker run -u root --rm \
  -v "$(pwd)/certs:/certs" \
  raul5589/logstash-custom:1.2-kafka \
  /usr/share/logstash/jdk/bin/keytool \
    -importcert \
    -file /certs/kafka-ca.crt \
    -alias kafka-ca \
    -keystore /certs/logstash-truststore.p12 \
    -storetype PKCS12 \
    -storepass "changeit" \
    -noprompt
```

### 2.3 Выставить правильные права на truststore

```bash
sudo chown 1000:1000 certs/logstash-truststore.p12
sudo chmod 644 certs/logstash-truststore.p12
```

### 2.4 Проверить содержимое truststore

В truststore должны быть два сертификата:

```bash
sudo docker run -u root --rm \
  -v "$(pwd)/certs:/certs" \
  raul5589/logstash-custom:1.2-kafka \
  /usr/share/logstash/jdk/bin/keytool \
    -list \
    -keystore /certs/logstash-truststore.p12 \
    -storetype PKCS12 \
    -storepass "changeit"
```

Ожидаемый вывод — две записи: одна с alias `1` (ELK-CA) и одна с alias `kafka-ca`.

---

## Шаг 3 — Распределение сертификатов

### 3.1 Выставить права на certs/

```bash
cd ~/elk
sudo chmod -R 644 certs/
sudo chmod 755 certs/
```

### 3.2 Раздать файлы на elk-node-02 и elk-node-03

**На elk-node-01** — запустить HTTP-сервер:

```bash
cd ~/elk/certs
python3 -m http.server 8888
```

**На elk-node-02 и elk-node-03** — скачать нужные файлы  
(`ca.key` и `elastic-stack-ca.p12` **не копировать**):

```bash
mkdir -p ~/elk/certs && cd ~/elk/certs

wget http://<IP_ELK_NODE_01>:8888/ca.crt
wget http://<IP_ELK_NODE_01>:8888/elastic-certificates.p12
wget http://<IP_ELK_NODE_01>:8888/kibana.crt
wget http://<IP_ELK_NODE_01>:8888/kibana.key
wget http://<IP_ELK_NODE_01>:8888/logstash.crt
wget http://<IP_ELK_NODE_01>:8888/logstash.key
wget http://<IP_ELK_NODE_01>:8888/logstash-truststore.p12
wget http://<IP_ELK_NODE_01>:8888/kafka-ca.crt
wget http://<IP_ELK_NODE_01>:8888/kafka.crt
wget http://<IP_ELK_NODE_01>:8888/kafka.key
```

После скачивания — остановить сервер на elk-node-01 (`Ctrl+C`).

> **`ca.key` и `elastic-stack-ca.p12` хранить только на elk-node-01 в безопасном месте.**

---

## Шаг 4 — Настройка .env

На **каждом** ELK-узле:

```bash
cd ~/elk
cp .env.template .env
nano .env
```

| Переменная                | Описание                                        | Пример                                             |
|---------------------------|-------------------------------------------------|----------------------------------------------------|
| `NODE_NAME`               | Уникальное имя ноды ES                          | `es-node-01` / `es-node-02` / `es-node-03`         |
| `PUBLISH_HOST`            | Локальный IP **этого** сервера                  | `192.168.1.20`                                     |
| `CLUSTER_NAME`            | Имя кластера ES (одинаковое на всех узлах)      | `elk-cluster`                                      |
| `SEED_HOSTS`              | IP всех ES-узлов через запятую                  | `192.168.1.20,192.168.1.21,192.168.1.22`           |
| `INITIAL_MASTER_NODES`    | Имена всех master-нод (одинаковое везде)        | `es-node-01,es-node-02,es-node-03`                 |
| `ELASTIC_PASSWORD`        | Пароль пользователя `elastic`                   | сложный пароль                                     |
| `KIBANA_PASSWORD`         | Пароль пользователя `kibana_system`             | сложный пароль                                     |
| `KIBANA_ENCRYPTION_KEY`   | Ключ шифрования Kibana (минимум 32 символа)     | случайная строка ≥ 32 символа                      |
| `S3_HOST`                 | Хост S3 для снапшотов                           | `s3.example.com`                                   |
| `KAFKA_BOOTSTRAP_SERVERS` | Брокеры Kafka — **порт 9092 (SSL)**             | `10.0.0.10:9092,10.0.0.11:9092,10.0.0.12:9092`    |

> `CLUSTER_NAME`, `SEED_HOSTS` и `INITIAL_MASTER_NODES` должны быть **одинаковыми** на всех трёх ELK-узлах.  
> `KAFKA_BOOTSTRAP_SERVERS` — обязательно порт **9092**, не 9094.

---

## Шаг 5 — Запуск кластера

> **Запускать одновременно на всех трёх узлах.**  
> Elasticsearch не достигнет кворума, пока не поднимутся все ноды из `INITIAL_MASTER_NODES`.

```bash
cd ~/elk
docker compose up -d
```

Проверить запуск:

```bash
docker compose ps
docker logs es-node-01 --tail 80
```

Дождаться строки в логах:

```
[INFO ][o.e.c.r.ClusterBootstrapService] master node changed ...
```

---

## Шаг 6 — Установка пароля kibana\_system

Выполнить **один раз** с любого ELK-узла после старта Elasticsearch.  
Подставить реальные значения паролей из `.env`. Можно воспользоваться командой `source .env`, чтобы пароли автоматически подтягивались:

```bash
curl -k -X POST "https://127.0.0.1:9200/_security/user/kibana_system/_password" \
-u "elastic:${ELASTIC_PASSWORD}" \
-H "Content-Type: application/json" \
-d "{ \"password\": \"${KIBANA_PASSWORD}\" }"
```

Успешный ответ: `{}`

> После этого перезапустить Kibana, чтобы она подхватила новый пароль:
> ```bash
> docker compose restart kibana
> ```

---

## Шаг 7 — Настройка ILM (Index Lifecycle Management)

> Logstash по умолчанию включает ILM. Без шаблона данные не попадут в индексы `filebeat-YYYY.MM.dd`.  
> Выполнить один раз с любого ELK-узла.

### 7.1 Создать ILM-политику

```bash
curl -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "https://localhost:9200/_ilm/policy/filebeat-policy" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "min_age": "0ms",
          "actions": {
            "rollover": {
              "max_age": "1d",
              "max_size": "10gb"
            }
          }
        },
        "warm": {
          "min_age": "7d",
          "actions": {
            "forcemerge": {
              "max_num_segments": 1
            },
            "shrink": {
              "number_of_shards": 1
            }
          }
        },
        "delete": {
          "min_age": "30d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'
```

### 7.2 Создать шаблон индекса

```bash
curl -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "https://localhost:9200/_index_template/filebeat-template" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["filebeat-*"],
    "template": {
      "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 1,
        "index.lifecycle.name": "filebeat-policy",
        "index.lifecycle.rollover_alias": "filebeat"
      },
      "mappings": {
        "dynamic": true,
        "properties": {
          "@timestamp": { "type": "date" },
          "message":    { "type": "text" },
          "host":       { "type": "object" },
          "tags":       { "type": "keyword" }
        }
      }
    },
    "priority": 100
  }'
```

### 7.3 Проверить ILM

```bash
# Список политик
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_ilm/policy?pretty"

# Статус ILM по индексам filebeat-*
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/filebeat-*/_ilm/explain?pretty"
```

---

## Шаг 8 — Настройка S3 snapshot-репозитория

### 8.1 Добавить S3-credentials в keystore Elasticsearch

Выполнить на **каждом** ELK-узле:

```bash
# Войти в контейнер
docker exec -it es-node-01 bash

# Добавить Access Key
echo "YOUR_S3_ACCESS_KEY" | \
  bin/elasticsearch-keystore add --stdin s3.client.default.access_key

# Добавить Secret Key
echo "YOUR_S3_SECRET_KEY" | \
  bin/elasticsearch-keystore add --stdin s3.client.default.secret_key

exit
```

Перезагрузить keystore без перезапуска контейнера:

```bash
curl -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -X POST "https://localhost:9200/_nodes/reload_secure_settings" \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 8.2 Зарегистрировать S3-репозиторий

```bash
curl -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "https://localhost:9200/_snapshot/s3_backup" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "YOUR_BUCKET_NAME",
      "base_path": "elasticsearch-snapshots",
      "client": "default"
    }
  }'
```

### 8.3 Проверить репозиторий

```bash
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_snapshot/s3_backup?pretty"
```

### 8.4 Тестовый снапшот вручную

```bash
curl -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "https://localhost:9200/_snapshot/s3_backup/snapshot_test?wait_for_completion=true" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": "filebeat-*",
    "ignore_unavailable": true,
    "include_global_state": false
  }'
```

### 8.5 Автоматические снапшоты (SLM)

```bash
curl -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -X PUT "https://localhost:9200/_slm/policy/daily-snapshots" \
  -H "Content-Type: application/json" \
  -d '{
    "schedule": "0 30 1 * * ?",
    "name": "<filebeat-snap-{now/d}>",
    "repository": "s3_backup",
    "config": {
      "indices": ["filebeat-*"],
      "ignore_unavailable": true,
      "include_global_state": false
    },
    "retention": {
      "expire_after": "30d",
      "min_count": 5,
      "max_count": 50
    }
  }'
```

---

## Шаг 9 — Проверка системы

### Health кластера

```bash
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_cluster/health?pretty"
```

Ожидаемый статус: `green` (все реплики размещены) или `yellow` (ждут узла).

### Узлы кластера

```bash
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_cat/nodes?v"
```

### Kibana

Открыть в браузере: `https://<IP>:5601`  
Логин: `elastic` / пароль из `ELASTIC_PASSWORD`

> Если браузер показывает ошибку сертификата — это ожидаемо для самоподписанного сертификата.  
> В **Firefox**: нажать "Дополнительно" → "Принять риск и продолжить".  
> В **Chrome/Edge**: напечатать на странице с ошибкой слово `thisisunsafe` (без поля ввода).

---

## ОБЯЗАТЕЛЬНОЕ Изменение конфигурации Logstash pipeline

Встроенный в образ конфиг `/usr/share/logstash/pipeline/logstash.conf` изменить, на приведенный ниже, поскольку конфиг в образе не использует SSL:

```ruby
input {
  kafka {
    bootstrap_servers => "${KAFKA_BOOTSTRAP_SERVERS}"
    topics => ["kafka-logs"]
    group_id => "logstash-diploma-v2"
    auto_offset_reset => "earliest"
    codec => "json"
    
    security_protocol => "SSL"
    ssl_truststore_type => "pkcs12"
    ssl_truststore_location => "/usr/share/logstash/config/certs/logstash-truststore.p12"
    ssl_truststore_password => "changeit"
    ssl_endpoint_identification_algorithm => ""
  }
}

filter {
}

output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    user => "elastic"
    password => "${ELASTIC_PASSWORD}"
    ssl_verification_mode => "none"
    index => "filebeat-%{+YYYY.MM.dd}"
  }
}
```

### Переопределить конфиг через volume mount (рекомендуется)

В `docker-compose.yml` раскомментировать строку в секции `logstash.volumes`:

```yaml
volumes:
  - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
```

Создать `~/elk/logstash.conf` с нужным содержимым, затем:

```bash
docker compose up -d --force-recreate logstash
```

### Проверить поступление данных из Kafka

```bash
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_cat/indices?v&index=filebeat-*"
```

В Kibana: **Stack Management → Index Management** → убедиться что индексы `filebeat-*` существуют.  
Затем: **Discover → Create data view → Pattern: `filebeat-*`**.

### Временное обновление без пересоздания контейнера

```bash
docker cp ~/elk/logstash.conf \
  logstash-es-node-01:/usr/share/logstash/pipeline/logstash.conf

docker restart logstash-es-node-01
```

> При следующем `docker compose up` конфиг вернётся к встроенному в образ (если volume не раскомментирован).

---

## Полезные команды

```bash
# Логи контейнеров
docker logs es-node-01          -f --tail 100
docker logs logstash-es-node-01 -f --tail 100
docker logs kibana-es-node-01   -f --tail 100

# Остановка (данные сохраняются в volumes)
docker compose down

# Запуск
docker compose up -d

# Список индексов по размеру
curl -k -u elastic:ELASTIC_PASSWORD \
  "https://localhost:9200/_cat/indices?v&s=store.size:desc"

# Состояние шардов
curl -k -u elastic:ELASTIC_PASSWORD \
  "https://localhost:9200/_cat/shards?v"
```

> ⚠️ **Никогда не выполняйте `docker compose down -v` без предварительного снапшота** — это уничтожит все данные Elasticsearch.

---

## Устранение неполадок

| Симптом | Причина | Решение |
|---------|---------|---------|
| ES не стартует: `max virtual memory areas` | `vm.max_map_count` слишком мал | `sudo sysctl -w vm.max_map_count=262144` |
| Кластер не собирается, статус `red` | Не все ноды запустились одновременно | Убедиться что все три ноды запущены, перезапустить |
| Kibana: `unable to authenticate kibana_system` | Пароль `kibana_system` не установлен | Выполнить команду из Шага 6 |
| Logstash: `Failed to create new NetworkClient` | Неверный порт в `KAFKA_BOOTSTRAP_SERVERS` | Убедиться что порт **9092**, не 9094 |
| Logstash: `Failed to construct kafka consumer` | Logstash не доверяет сертификату Kafka | Выполнить Шаг 2 — добавить Kafka-CA в truststore |
| Индексы `filebeat-*` не создаются | ILM-шаблон не применён | Выполнить Шаг 7 |
| Logstash: `index write blocked` | ILM-политика заблокировала индекс | Проверить `/_ilm/explain` |
| S3: `repository_exception` | Неверные credentials или endpoint | Проверить keystore и `S3_HOST` в `.env` |
| Kibана не открывается в браузере | Кеш старого сертификата | Открыть в режиме инкогнито или очистить кеш сертификатов |
