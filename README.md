# Общее описание
Прототип обмена 1с с Битрикс24 через брокера сообщений Kafka. 1с делает http-запрос в сервис `kafka-rest`, который кладет полученное сообщение в топик `EVENTS_1C_UT`.
Этот топик читает jdbc-коннектор и сохраняет все сообщения в базу данных Битрикс24. Результат обработки Битрикс24 кладет в mysql-таблицу RESULTS_1C_UT, которую также читает коннектор и наполняет им соответствующий топик RESULTS_1C_UT.
Помимо этого, на каждый топик создается efk-коннектор, который сохраняем все сообщения в EFK-стек для удобного просмотра, фильтрации, агрегации.

![Схема работы](/assets/img/schema.png)

Стек (docker-compose):
- `kafka`: брокер сообщений с топиками EVENTS_1C_UT и RESULTS_1C_UT
- `kafka-rest`: собственное stateless java приложений, которое принимает http-запрос и кладет его body в топик. Слушает адреса вида http://localhost:8080/v1/<topic_name>
- `registry`: собственный локальный docker-hub для хранения образа kafka-rest
- `schema registry`: хранит схемы сообщений (в нашем случае всего одна универсальная схема).
- `kafka-connect`: связывает топик kafka и внешние системы. Например читает топик и записывает все сообщения в mysql таблицу или читает mysql-таблицу и помещает все строки в топик kafka
- `ELK`: хранит все сообщения kafka (события 1с, события Б24, результаты обработок и т.д.). Хранит с целью удобного просмотре сообщений (парсит их, позволят фильтровать по полям и т.д.). Сами сообщения все-равно продолжают хранится в kafka и она имеет собственные правила их ротации

С целью универсальности, обмен осуществляется лишь одним типом сообщения GeneralMessage, содержащий одно поле `data` типа `string` (json строка).
Его avro-схема GeneralMessage.avsc:
```json
{
  "namespace": "ru.mironovmike.kafkarest.schema",
  "type": "record",
  "name": "GeneralMessage",
  "fields": [
    {"name": "data", "type": "string"}
  ]
}
```

При сохранении сообщений в ELK, оно проходит препроцессинг (ingest pipeline) с двумя процессорами. Первый преобразует строку в json-объект, а второй удаляет ключ `data`. Описание Ingest Pipeline:
```json
[
  {
    "json": {
      "field": "data",
      "add_to_root": true
    }
  },
  {
    "remove": {
      "field": "data",
      "ignore_missing": true
    }
  }
]
```

Настройки index template для применения этого pipeline:
```json
{
  "index": {
    "default_pipeline": "data_field_to_json_with_remove"
  }
}
```

Пример маппинга полей в ELK:
```json
{
  "_routing": {
    "required": false
  },
  "dynamic": false,
  "_source": {
    "excludes": [],
    "includes": [],
    "enabled": true
  },
  "dynamic_templates": [],
  "properties": {
    "contact_informations": {
      "type": "nested",
      "properties": {
        "kind": {
          "type": "keyword"
        },
        "type": {
          "type": "keyword"
        },
        "value": {
          "type": "text"
        }
      }
    },
    "patronymic": {
      "type": "keyword"
    },
    "partner": {
      "type": "keyword"
    },
    "post": {
      "type": "text"
    },
    "entityType": {
      "type": "keyword"
    },
    "name": {
      "type": "keyword"
    },
    "guid": {
      "type": "keyword"
    },
    "last_name": {
      "type": "keyword"
    }
  }
}
```


# .m2/settings.xml example
```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
	<profiles>
		<profile>
			<id>default</id>
			<activation>
				<activeByDefault>true</activeByDefault>
			</activation>
			<properties>
				<docker-registry>alpha.mironovmike.ru:5000</docker-registry>
				<docker-registry.user>your_registry_user</docker-registry.user>
				<docker-registry.password>your_registry_password</docker-registry.password>
			</properties>
		</profile>
	</profiles>
</settings>
```

# Управление Kafka
bin path
`/opt/bitnami/kafka/bin/`

Получить список тем
`kafka-topics.sh --bootstrap-server kafka:9092 --list`

Запуск console-consumer
`kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic EVENTS_1C_UT --from-beginning`

# Docker registry
Получить список репозиториев
`GET http://alpha.local:5000/v2/_catalog`

Получить список тегов конкретного репозитория
`GET http://alpha.local:5000/v2/kafka-rest/tags/list`

# Schema registry
Получить список схем
`GET http://alpha.local:8081/schemas`

# Kafka-connect
Запустить jdbc sink
```
POST http://alpha.local:8083/connectors
{
    "name": "1c-events-to-b24-connector",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
        "tasks.max": 1,
        "topics": "EVENTS_1C_UT",
        "connection.url": "jdbc:mysql://mysql:3306/b24?ssl-mode=SSL_MODE",
        "connection.user": "bitrix",
        "connection.password": "bitrix",
        "auto.create": "true",
        "auto.evolve": "true",
        "delete.enabled": "false",
        "pk.mode": "none",
        "pk.fields": "id"
    }
}
```

Запустить EFK-sink
```
POST http://alpha.local:8083/connectors
{
    "name": "1c-events-to-elasticsearch-connector",
    "config": {
        "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
        "tasks.max": 1,
        "topics": "EVENTS_1C_UT",
        "connection.url": "http://elasticsearch:9200",
        "key.ignore": "true",
        "schema.ignore": "true",
        "drop.invalid.message": "true",
        "behavior.on.malformed.documents": "IGNORE"
    }
}
```

Пример сообщения, отправляемое 1с в `kafka-rest`:
```
POST http://localhost:8080/v1/EVENTS_1C_UT
{
    "entityType": "ContactPersons",
    "guid": "50d66b63-3624-471b-8625-e5c67b413e21",
    "partner": "8142482b-02ec-11e5-940d-e0db550d5514",
    "name": "Дарья",
    "patronymic": "Игоревна",
    "last_name": "Федорова",
    "post": "Менеджер",
    "contact_informations": [
        {
            "type": "Телефон",
            "kind": "e6548615-5ed5-11e4-9409-e0db550d5514",
            "value": "+79991234567"
        },
        {
            "type": "Телефон",
            "kind": "e6548615-5ed5-11e4-9409-e0db550d5514",
            "value": "+79999999999"
        }
    ]
}
```