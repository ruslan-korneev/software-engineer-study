# Загрузка данных в Elasticsearch

## CRUD операции

Elasticsearch предоставляет RESTful API для всех операций с документами. Каждый документ идентифицируется комбинацией `index/_doc/id`.

## Создание документов (Create)

### Индексация с указанием ID

```bash
# PUT /{index}/_doc/{id}
curl -X PUT "localhost:9200/products/_doc/1" -H 'Content-Type: application/json' -d'
{
  "name": "iPhone 15 Pro",
  "price": 999.99,
  "category": "smartphones",
  "in_stock": true,
  "created_at": "2024-01-15T10:30:00Z"
}'
```

**Ответ:**
```json
{
  "_index": "products",
  "_id": "1",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
```

### Индексация с автогенерацией ID

```bash
# POST /{index}/_doc
curl -X POST "localhost:9200/products/_doc" -H 'Content-Type: application/json' -d'
{
  "name": "Samsung Galaxy S24",
  "price": 899.99,
  "category": "smartphones"
}'
```

**Ответ содержит сгенерированный ID:**
```json
{
  "_index": "products",
  "_id": "abc123xyz789",
  "_version": 1,
  "result": "created"
}
```

### Create only (без перезаписи)

```bash
# Создать только если документ не существует
curl -X PUT "localhost:9200/products/_create/1" -H 'Content-Type: application/json' -d'
{
  "name": "Product 1"
}'

# Альтернативный синтаксис
curl -X PUT "localhost:9200/products/_doc/1?op_type=create" -H 'Content-Type: application/json' -d'
{
  "name": "Product 1"
}'
```

**Если документ уже существует — ошибка 409 Conflict:**
```json
{
  "error": {
    "type": "version_conflict_engine_exception",
    "reason": "[1]: version conflict, document already exists"
  },
  "status": 409
}
```

## Чтение документов (Read)

### Получение одного документа

```bash
# GET /{index}/_doc/{id}
curl -X GET "localhost:9200/products/_doc/1"
```

**Ответ:**
```json
{
  "_index": "products",
  "_id": "1",
  "_version": 1,
  "_seq_no": 0,
  "_primary_term": 1,
  "found": true,
  "_source": {
    "name": "iPhone 15 Pro",
    "price": 999.99,
    "category": "smartphones"
  }
}
```

### Получение только source

```bash
# Только _source без метаданных
curl -X GET "localhost:9200/products/_source/1"

# Выбор конкретных полей
curl -X GET "localhost:9200/products/_doc/1?_source=name,price"

# Исключение полей
curl -X GET "localhost:9200/products/_doc/1?_source_excludes=description"
```

### Multi Get API

```bash
# Получение нескольких документов за один запрос
curl -X GET "localhost:9200/_mget" -H 'Content-Type: application/json' -d'
{
  "docs": [
    { "_index": "products", "_id": "1" },
    { "_index": "products", "_id": "2" },
    { "_index": "users", "_id": "100" }
  ]
}'

# Если все документы из одного индекса
curl -X GET "localhost:9200/products/_mget" -H 'Content-Type: application/json' -d'
{
  "ids": ["1", "2", "3"]
}'
```

### Проверка существования

```bash
# HEAD запрос — только проверка (без тела ответа)
curl -I "localhost:9200/products/_doc/1"

# 200 OK — существует
# 404 Not Found — не существует
```

## Обновление документов (Update)

### Полная замена документа

```bash
# PUT заменяет документ полностью
curl -X PUT "localhost:9200/products/_doc/1" -H 'Content-Type: application/json' -d'
{
  "name": "iPhone 15 Pro Max",
  "price": 1199.99,
  "category": "smartphones",
  "in_stock": true
}'
```

**result: "updated", _version увеличивается:**
```json
{
  "_id": "1",
  "_version": 2,
  "result": "updated"
}
```

### Частичное обновление (Update API)

```bash
# POST /{index}/_update/{id}
curl -X POST "localhost:9200/products/_update/1" -H 'Content-Type: application/json' -d'
{
  "doc": {
    "price": 949.99,
    "discount": 10
  }
}'
```

### Обновление с помощью скрипта

```bash
# Увеличить цену на 10%
curl -X POST "localhost:9200/products/_update/1" -H 'Content-Type: application/json' -d'
{
  "script": {
    "source": "ctx._source.price *= params.factor",
    "lang": "painless",
    "params": {
      "factor": 1.1
    }
  }
}'

# Добавить элемент в массив
curl -X POST "localhost:9200/products/_update/1" -H 'Content-Type: application/json' -d'
{
  "script": {
    "source": "ctx._source.tags.add(params.tag)",
    "params": {
      "tag": "bestseller"
    }
  }
}'

# Условное обновление
curl -X POST "localhost:9200/products/_update/1" -H 'Content-Type: application/json' -d'
{
  "script": {
    "source": "if (ctx._source.in_stock == true) { ctx._source.quantity -= params.sold; if (ctx._source.quantity <= 0) { ctx._source.in_stock = false; } } else { ctx.op = \"noop\"; }",
    "params": {
      "sold": 5
    }
  }
}'
```

### Upsert (update or insert)

```bash
curl -X POST "localhost:9200/products/_update/999" -H 'Content-Type: application/json' -d'
{
  "doc": {
    "name": "New Product",
    "views": 1
  },
  "doc_as_upsert": true
}'

# Или с разными значениями для insert и update
curl -X POST "localhost:9200/products/_update/999" -H 'Content-Type: application/json' -d'
{
  "script": {
    "source": "ctx._source.views += 1"
  },
  "upsert": {
    "name": "New Product",
    "views": 1
  }
}'
```

### Update by Query

```bash
# Обновить все документы, соответствующие запросу
curl -X POST "localhost:9200/products/_update_by_query" -H 'Content-Type: application/json' -d'
{
  "query": {
    "term": {
      "category": "smartphones"
    }
  },
  "script": {
    "source": "ctx._source.category = \"mobile_phones\"",
    "lang": "painless"
  }
}'

# С параметрами
curl -X POST "localhost:9200/products/_update_by_query?conflicts=proceed&refresh=true" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "script": {
    "source": "ctx._source.updated_at = params.now",
    "params": {
      "now": "2024-01-15T12:00:00Z"
    }
  }
}'
```

## Удаление документов (Delete)

### Удаление по ID

```bash
curl -X DELETE "localhost:9200/products/_doc/1"
```

**Ответ:**
```json
{
  "_index": "products",
  "_id": "1",
  "_version": 3,
  "result": "deleted",
  "_shards": {
    "total": 2,
    "successful": 2,
    "failed": 0
  }
}
```

### Delete by Query

```bash
# Удалить все документы, соответствующие запросу
curl -X POST "localhost:9200/products/_delete_by_query" -H 'Content-Type: application/json' -d'
{
  "query": {
    "range": {
      "created_at": {
        "lt": "2023-01-01"
      }
    }
  }
}'

# Удалить все документы из индекса (быстрее удалить индекс)
curl -X POST "localhost:9200/products/_delete_by_query?conflicts=proceed" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  }
}'
```

## Bulk API

Bulk API позволяет выполнять множество операций за один HTTP-запрос, что значительно повышает производительность.

### Формат Bulk запроса

```
action_and_meta_data\n
optional_source\n
action_and_meta_data\n
optional_source\n
...
```

### Пример Bulk запроса

```bash
curl -X POST "localhost:9200/_bulk" -H 'Content-Type: application/x-ndjson' -d'
{"index": {"_index": "products", "_id": "1"}}
{"name": "Product 1", "price": 100}
{"index": {"_index": "products", "_id": "2"}}
{"name": "Product 2", "price": 200}
{"create": {"_index": "products", "_id": "3"}}
{"name": "Product 3", "price": 300}
{"update": {"_index": "products", "_id": "1"}}
{"doc": {"price": 150}}
{"delete": {"_index": "products", "_id": "2"}}
'
```

**ВАЖНО:**
- Каждая строка должна заканчиваться `\n` (включая последнюю)
- Content-Type: `application/x-ndjson` или `application/json`

### Bulk для одного индекса

```bash
curl -X POST "localhost:9200/products/_bulk" -H 'Content-Type: application/x-ndjson' -d'
{"index": {"_id": "1"}}
{"name": "Product 1", "price": 100}
{"index": {"_id": "2"}}
{"name": "Product 2", "price": 200}
{"index": {"_id": "3"}}
{"name": "Product 3", "price": 300}
'
```

### Ответ Bulk API

```json
{
  "took": 30,
  "errors": false,
  "items": [
    {
      "index": {
        "_index": "products",
        "_id": "1",
        "_version": 1,
        "result": "created",
        "status": 201
      }
    },
    {
      "index": {
        "_index": "products",
        "_id": "2",
        "_version": 1,
        "result": "created",
        "status": 201
      }
    }
  ]
}
```

### Обработка ошибок в Bulk

```json
{
  "took": 30,
  "errors": true,
  "items": [
    {
      "index": {
        "_index": "products",
        "_id": "1",
        "status": 201,
        "result": "created"
      }
    },
    {
      "index": {
        "_index": "products",
        "_id": "2",
        "status": 400,
        "error": {
          "type": "mapper_parsing_exception",
          "reason": "failed to parse field [price] of type [float]"
        }
      }
    }
  ]
}
```

### Bulk с Python

```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk, streaming_bulk

es = Elasticsearch(["http://localhost:9200"])

# Генератор документов
def generate_actions():
    for i in range(10000):
        yield {
            "_index": "products",
            "_id": str(i),
            "_source": {
                "name": f"Product {i}",
                "price": i * 10.0,
                "category": "electronics"
            }
        }

# Простой bulk
success, failed = bulk(
    es,
    generate_actions(),
    chunk_size=1000,
    max_retries=3,
    raise_on_error=False
)
print(f"Indexed {success} documents, {failed} failed")

# Streaming bulk с обработкой ошибок
for ok, result in streaming_bulk(
    es,
    generate_actions(),
    chunk_size=1000,
    raise_on_error=False
):
    if not ok:
        print(f"Failed: {result}")
```

## Оптимизация индексации

### Настройки для массовой загрузки

```bash
# Отключить refresh на время загрузки
curl -X PUT "localhost:9200/products/_settings" -H 'Content-Type: application/json' -d'
{
  "index": {
    "refresh_interval": "-1"
  }
}'

# Увеличить количество реплик после загрузки
curl -X PUT "localhost:9200/products/_settings" -H 'Content-Type: application/json' -d'
{
  "index": {
    "number_of_replicas": 0
  }
}'

# После загрузки — вернуть настройки
curl -X PUT "localhost:9200/products/_settings" -H 'Content-Type: application/json' -d'
{
  "index": {
    "refresh_interval": "1s",
    "number_of_replicas": 1
  }
}'

# Принудительный refresh
curl -X POST "localhost:9200/products/_refresh"

# Force merge для оптимизации сегментов
curl -X POST "localhost:9200/products/_forcemerge?max_num_segments=1"
```

### Параметры Bulk запросов

| Параметр | Описание | Рекомендация |
|----------|----------|--------------|
| **chunk_size** | Документов в одном запросе | 1000-5000 |
| **refresh** | Обновить после каждого bulk | `false` для загрузки |
| **timeout** | Таймаут операции | `5m` для больших bulk |
| **pipeline** | Ingest pipeline для обработки | По необходимости |

## Ingest Pipelines

Ingest pipelines позволяют обрабатывать документы перед индексацией.

### Создание pipeline

```bash
curl -X PUT "localhost:9200/_ingest/pipeline/my-pipeline" -H 'Content-Type: application/json' -d'
{
  "description": "Process incoming documents",
  "processors": [
    {
      "set": {
        "field": "indexed_at",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "lowercase": {
        "field": "category"
      }
    },
    {
      "trim": {
        "field": "name"
      }
    },
    {
      "remove": {
        "field": "internal_field",
        "ignore_missing": true
      }
    }
  ]
}'
```

### Распространённые процессоры

```bash
curl -X PUT "localhost:9200/_ingest/pipeline/enrichment-pipeline" -H 'Content-Type: application/json' -d'
{
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": ["%{IP:client_ip} %{WORD:method} %{URIPATHPARAM:request}"]
      }
    },
    {
      "geoip": {
        "field": "client_ip",
        "target_field": "geo"
      }
    },
    {
      "user_agent": {
        "field": "user_agent_string",
        "target_field": "user_agent"
      }
    },
    {
      "date": {
        "field": "timestamp_string",
        "target_field": "@timestamp",
        "formats": ["yyyy-MM-dd HH:mm:ss", "ISO8601"]
      }
    },
    {
      "script": {
        "source": "ctx.price_category = ctx.price > 1000 ? 'expensive' : 'affordable';"
      }
    },
    {
      "rename": {
        "field": "old_name",
        "target_field": "new_name"
      }
    },
    {
      "split": {
        "field": "tags",
        "separator": ","
      }
    }
  ]
}'
```

### Использование pipeline

```bash
# При индексации
curl -X POST "localhost:9200/products/_doc?pipeline=my-pipeline" -H 'Content-Type: application/json' -d'
{
  "name": "  New Product  ",
  "category": "ELECTRONICS"
}'

# Pipeline по умолчанию для индекса
curl -X PUT "localhost:9200/products/_settings" -H 'Content-Type: application/json' -d'
{
  "index.default_pipeline": "my-pipeline"
}'

# В bulk операциях
curl -X POST "localhost:9200/_bulk?pipeline=my-pipeline" -H 'Content-Type: application/x-ndjson' -d'
{"index": {"_index": "products"}}
{"name": "Product 1"}
'
```

### Тестирование pipeline

```bash
curl -X POST "localhost:9200/_ingest/pipeline/my-pipeline/_simulate" -H 'Content-Type: application/json' -d'
{
  "docs": [
    {
      "_source": {
        "name": "  Test Product  ",
        "category": "ELECTRONICS"
      }
    }
  ]
}'
```

## Reindex API

```bash
# Копирование между индексами
curl -X POST "localhost:9200/_reindex" -H 'Content-Type: application/json' -d'
{
  "source": {
    "index": "old_products"
  },
  "dest": {
    "index": "new_products"
  }
}'

# С выборкой документов
curl -X POST "localhost:9200/_reindex" -H 'Content-Type: application/json' -d'
{
  "source": {
    "index": "products",
    "query": {
      "term": {
        "category": "electronics"
      }
    }
  },
  "dest": {
    "index": "electronics_products"
  }
}'

# С трансформацией через script
curl -X POST "localhost:9200/_reindex" -H 'Content-Type: application/json' -d'
{
  "source": {
    "index": "old_products"
  },
  "dest": {
    "index": "new_products"
  },
  "script": {
    "source": "ctx._source.migrated = true; ctx._source.price *= 1.1"
  }
}'

# Reindex из remote кластера
curl -X POST "localhost:9200/_reindex" -H 'Content-Type: application/json' -d'
{
  "source": {
    "remote": {
      "host": "http://remote-es:9200"
    },
    "index": "products"
  },
  "dest": {
    "index": "products"
  }
}'
```

## Best Practices

### Оптимизация загрузки данных

```
✅ Используйте Bulk API вместо одиночных запросов
✅ Оптимальный размер bulk: 5-15 MB
✅ Отключайте refresh_interval при массовой загрузке
✅ Уменьшайте replicas до 0 на время загрузки
✅ Используйте _routing для локализации данных
✅ Применяйте ingest pipelines для предобработки
```

### Параметры производительности

```python
# Рекомендуемые настройки для bulk
BULK_CONFIG = {
    "chunk_size": 1000,          # документов в batch
    "max_chunk_bytes": 10485760,  # 10 MB
    "thread_count": 4,            # параллельные потоки
    "queue_size": 4,              # размер очереди
}

# Расчёт оптимального batch size
# bulk_size = total_size / time_limit
# Пример: 1GB / 10min = ~1.7 MB/s -> batch 1000 docs @ 1.5KB avg
```

### Обработка ошибок

```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk

def safe_bulk_index(es, actions):
    """Bulk indexing с обработкой ошибок."""
    success = 0
    failed = []

    for ok, result in bulk(
        es,
        actions,
        chunk_size=1000,
        raise_on_error=False,
        raise_on_exception=False
    ):
        if ok:
            success += 1
        else:
            failed.append(result)

    if failed:
        # Логировать ошибки
        for item in failed:
            print(f"Failed: {item}")

        # Retry с экспоненциальной задержкой
        # или сохранить для последующей обработки

    return success, failed
```

### Версионирование

```bash
# Optimistic concurrency control
curl -X PUT "localhost:9200/products/_doc/1?if_seq_no=10&if_primary_term=1" -d'
{
  "name": "Updated Product"
}'

# Если seq_no изменился — 409 Conflict
# Нужно перечитать документ и повторить операцию
```
