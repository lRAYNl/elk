# ELK Stack — Elasticsearch + Logstash + Kibana

Развёртывание трёхузлового кластера **Elasticsearch 9.2.2** с Logstash и Kibana.  
Транспорт и HTTP защищены TLS. Логи поступают из Kafka через Logstash.
Для установки git clone https://github.com/lRAYNl/elk.git
---

## Содержание

- [Архитектура](#архитектура)
- [Требования](#требования)
- [Структура репозитория](#структура-репозитория)
- [Шаг 1 — Генерация SSL-сертификатов](#шаг-1--генерация-ssl-сертификатов)
- [Шаг 2 — Настройка .env](#шаг-2--настройка-env)
- [Шаг 3 — Запуск кластера](#шаг-3--запуск-кластера)
- [Шаг 4 — Установка пароля kibana\_system](#шаг-4--установка-пароля-kibana_system)
- [Шаг 5 — Настройка ILM (Index Lifecycle Management)](#шаг-5--настройка-ilm-index-lifecycle-management)
- [Шаг 6 — Настройка S3 snapshot-репозитория](#шаг-6--настройка-s3-snapshot-репозитория)
- [Шаг 7 — Проверка системы](#шаг-7--проверка-системы)
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

### Обязательная системная настройка (на каждом ELK-узле)

```bash
# На каждом ELK-узле
git clone https://github.com/lRAYNl/elk.git ~/elk

# На каждом Kafka-узле
git clone https://github.com/lRAYNl/kafka.git ~/kafka

# На каждом хосте, где собираются логи
git clone https://github.com/lRAYNl/filebeat.git ~/filebeat

# Обновить пакеты
sudo apt-get update && sudo apt-get upgrade -y

# Установить Docker
sudo apt install docker-compose-v2

# Применить немедленно
sudo sysctl -w vm.max_map_count=262144

# Сделать постоянным после перезагрузки
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
```

> Без этой настройки Elasticsearch не запустится.

---

## Структура репозитория

```
elk/
├── docker-compose.yml
├── .env.template
├── logstash.conf          # переопределение pipeline (см. раздел Logstash)
└── certs/                 # создаётся вручную (см. Шаг 1)
    ├── ca.crt
    ├── ca.key             # только на elk-node-01!
    ├── elastic-certificates.p12
    ├── elastic-stack-ca.p12
    ├── kibana.crt
    ├── kibana.key
    ├── logstash.crt
    ├── logstash.key
    ├── logstash-truststore.p12
    ├── kafka.crt          # нужен Logstash для SSL с Kafka
    └── kafka.key
```

---

## Шаг 1 — Генерация SSL-сертификатов

> Все команды выполняются на **elk-node-01** в директории `~/elk/certs/`.
> Затем файлы распределяются по остальным узлам.

```bash
mkdir -p ~/elk/certs && cd ~/elk/certs
```

### 1.1 Корневой CA

```bash
openssl genrsa -out ca.key 4096

openssl req -new -x509 -days 3650 \
  -key ca.key \
  -out ca.crt \
  -subj "/CN=ELK-CA/O=MyOrg/C=RU"
```

### 1.2 Сертификаты Elasticsearch (формат PKCS12)

Elasticsearch использует `.p12` для транспортного и HTTP TLS:

```bash
# Создать CA в формате p12 через elasticsearch-certutil
docker run --rm \
  -v "$(pwd):/certs" \
  docker.elastic.co/elasticsearch/elasticsearch:9.2.2 \
  bin/elasticsearch-certutil ca \
    --out /certs/elastic-stack-ca.p12 \
    --pass ""

# Создать сертификат узлов ES
docker run --rm \
  -v "$(pwd):/certs" \
  docker.elastic.co/elasticsearch/elasticsearch:9.2.2 \
  bin/elasticsearch-certutil cert \
    --ca /certs/elastic-stack-ca.p12 \
    --ca-pass "" \
    --out /certs/elastic-certificates.p12 \
    --pass ""
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

# Truststore для Logstash → Elasticsearch
openssl pkcs12 -export -nokeys \
  -in ca.crt \
  -out logstash-truststore.p12 \
  -passout pass:
```

### 1.5 Сертификат Kafka (для Logstash-клиента)

```bash
openssl genrsa -out kafka.key 2048

openssl req -new -key kafka.key -out kafka.csr \
  -subj "/CN=kafka/O=MyOrg/C=RU"

openssl x509 -req -days 3650 \
  -in kafka.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out kafka.crt
```

### 1.6 Распределить сертификаты

```bash
# Для начала выделяем права на папку (на всех узлах)

sudo chmod -R 644 certs/

sudo chmod 755 certs/

# На все остальные (ca.key и elastic-stack-ca.p12 НЕ копировать) перенести сертификаты удобным способом

cd certs/

sudo python3 -m http.server PORT #в port пишем свой порт

# На остальных нодах пишем

cd certs/

sudo wget http://IP:PORT/требуемый_файл

```

> **`ca.key` и `elastic-stack-ca.p12` хранить только на elk-node-01 в безопасном месте.**

---

## Шаг 2 — Настройка .env

На **каждом** ELK-узле:

```bash
cd ~/elk
cp .env.template .env
nano .env
```

| Переменная               | Описание                                        | Пример                                       |
|--------------------------|-------------------------------------------------|----------------------------------------------|
| `NODE_NAME`              | Уникальное имя ноды ES                          | `es-node-01`                                 |
| `PUBLISH_HOST`           | Локальный IP **этого** сервера                  | `192.168.1.20`                               |
| `CLUSTER_NAME`           | Имя кластера ES (одинаковое на всех узлах)      | `elk-cluster`                                |
| `SEED_HOSTS`             | IP всех ES-узлов через запятую                  | `192.168.1.20,192.168.1.21,192.168.1.22`     |
| `INITIAL_MASTER_NODES`   | Имена всех master-нод (одинаковое везде)        | `es-node-01,es-node-02,es-node-03`           |
| `ELASTIC_PASSWORD`       | Пароль пользователя `elastic`                   | сложный пароль                               |
| `KIBANA_PASSWORD`        | Пароль пользователя `kibana_system`             | сложный пароль                               |
| `KIBANA_ENCRYPTION_KEY`  | Ключ шифрования Kibana (минимум 32 символа)     | случайная строка ≥ 32 символа                |
| `S3_HOST`                | Хост S3 для снапшотов                           | `s3.example.com`                             |
| `KAFKA_BOOTSTRAP_SERVERS`| Брокеры Kafka для Logstash                      | `10.0.0.10:9092,10.0.0.11:9092,10.0.0.12:9092` |

> `CLUSTER_NAME`, `SEED_HOSTS` и `INITIAL_MASTER_NODES` должны быть **одинаковыми** на всех трёх ELK-узлах.

---

## Шаг 3 — Запуск кластера

> **Запускать одновременно на всех трёх узлах.** Elasticsearch не достигнет кворума, пока не поднимутся все три ноды из `INITIAL_MASTER_NODES`.

```bash
cd ~/elk
docker compose up -d
```

Проверить запуск:

```bash
docker compose ps
docker logs ${NODE_NAME} --tail 80
```

Дождаться строки в логах:

```
[INFO ][o.e.c.r.ClusterBootstrapService] [es-node-01] master node changed ...
```

---

## Шаг 4 — Установка пароля kibana\_system

Выполнить **один раз** с любого ELK-узла после старта Elasticsearch:

```bash
curl -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -X POST "https://localhost:9200/_security/user/kibana_system/_password" \
  -H "Content-Type: application/json" \
  -d '{"password": "${KIBANA_PASSWORD}"}'
```

Успешный ответ: `{}`

> Замените `${ELASTIC_PASSWORD}` и `${KIBANA_PASSWORD}` на значения из `.env`.

---

## Шаг 5 — Настройка ILM (Index Lifecycle Management)

Logstash по умолчанию включает ILM и переопределяет настройку `index`, из-за чего данные
не попадают в нужные индексы `filebeat-YYYY.MM.dd`. Необходимо создать ILM-политику и
шаблон индекса, которые соответствуют конфигу Logstash.

### 5.1 Создать ILM-политику

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

> Параметры политики (`max_age`, `min_age` фазы delete и т.д.) — настройте под свои требования.

### 5.2 Создать шаблон индекса

Шаблон применяется ко всем индексам по маске `filebeat-*` и привязывает к ним политику ILM:

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

### 5.3 Проверить ILM

```bash
# Список политик
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_ilm/policy?pretty"

# Статус ILM по индексам filebeat-*
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/filebeat-*/_ilm/explain?pretty"
```

---

## Шаг 6 — Настройка S3 snapshot-репозитория

### 6.1 Добавить S3-credentials в keystore Elasticsearch

Выполнить на **каждом** ELK-узле:

```bash
# Войти в контейнер
docker exec -it ${NODE_NAME} bash

# Добавить Access Key
echo "YOUR_S3_ACCESS_KEY" | \
  bin/elasticsearch-keystore add --stdin s3.client.default.access_key

# Добавить Secret Key
echo "YOUR_S3_SECRET_KEY" | \
  bin/elasticsearch-keystore add --stdin s3.client.default.secret_key

exit
```

После добавления credentials перезагрузить настройки безопасного хранилища:

```bash
curl -k \
  -u elastic:${ELASTIC_PASSWORD} \
  -X POST "https://localhost:9200/_nodes/reload_secure_settings" \
  -H "Content-Type: application/json" \
  -d '{}'
```

### 6.2 Зарегистрировать S3-репозиторий

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

### 6.3 Проверить репозиторий

```bash
curl -k \
  -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_snapshot/s3_backup?pretty"
```

### 6.4 Создать снапшот вручную (тест)

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

### 6.5 Настроить автоматические снапшоты через Snapshot Lifecycle Management (SLM)

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

## Шаг 7 — Проверка системы

### Health кластера

```bash
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_cluster/health?pretty"
```

Ожидаемый статус: `green` (все реплики размещены) или `yellow` (реплики ожидают узла).

### Узлы кластера

```bash
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_cat/nodes?v"
```

### Kibana

Открыть в браузере: `https://<IP>:5601`  
Логин: `elastic` / пароль из `ELASTIC_PASSWORD`

### Проверить поступление данных

```bash
# Появление индексов filebeat-*
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_cat/indices?v&index=filebeat-*"
```

В Kibana: **Stack Management → Index Management** — убедиться, что индексы `filebeat-*` существуют.  
Затем создать Data View: **Discover → Create data view → Pattern: `filebeat-*`**.

---

## Изменение конфигурации Logstash pipeline

Встроенный в образ конфиг `/usr/share/logstash/pipeline/logstash.conf`:

```ruby
input {
  kafka {
    bootstrap_servers => "${KAFKA_BOOTSTRAP_SERVERS}"
    topics => ["kafka-logs"]
    group_id => "logstash-diploma-v2"
    auto_offset_reset => "earliest"
    codec => "json"
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

### Переопределить конфиг через volume mount

В `docker-compose.yml` в секции `logstash.volumes` есть закомментированная строка. Раскомментировать её:

```yaml
volumes:
  - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
```

Создать файл `~/elk/logstash.conf` с нужным содержимым, затем пересоздать контейнер:

```bash
docker compose up -d --force-recreate logstash
```

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
docker logs es-node-01         -f --tail 100
docker logs logstash-es-node-01 -f --tail 100
docker logs kibana-es-node-01  -f --tail 100

# Остановка (данные сохраняются)
docker compose down

# Запуск
docker compose up -d

# Список индексов
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_cat/indices?v&s=store.size:desc"

# Состояние шардов
curl -k -u elastic:${ELASTIC_PASSWORD} \
  "https://localhost:9200/_cat/shards?v"
```

>  **Никогда не выполняйте `docker compose down -v` без предварительного снапшота** — это уничтожит все данные Elasticsearch.

---

## Устранение неполадок

| Симптом | Причина | Решение |
|---------|---------|---------|
| ES не стартует: `max virtual memory areas` | `vm.max_map_count` слишком мал | `sudo sysctl -w vm.max_map_count=262144` |
| Кластер не собирается, статус `red` | Не все ноды запустились одновременно | Убедиться, что все три ноды запущены |
| Kibana: `kibana_system password incorrect` | Пароль `kibana_system` не установлен | Выполнить команду из Шага 4 |
| Индексы `filebeat-*` не создаются | ILM-шаблон не применён | Выполнить Шаг 5 |
| Logstash: `index write blocked` | ILM-политика заблокировала индекс | Проверить фазу ILM: `/_ilm/explain` |
| S3: `repository_exception` при создании репо | Неверные credentials или endpoint | Проверить keystore и `S3_HOST` в `.env` |
| Logstash не подключается к Kafka | Неверный `KAFKA_BOOTSTRAP_SERVERS` | Проверить IP:порт Kafka-брокеров, доступность порта 9092 |
