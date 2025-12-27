# Продвинутые функции Elasticsearch

## Machine Learning

Elasticsearch Machine Learning позволяет обнаруживать аномалии и делать прогнозы на основе данных.

### Anomaly Detection Jobs

```bash
# Создание job для обнаружения аномалий
curl -X PUT "localhost:9200/_ml/anomaly_detectors/request_rate_anomaly" -H 'Content-Type: application/json' -d'
{
  "description": "Detect anomalies in request rate",
  "analysis_config": {
    "bucket_span": "15m",
    "detectors": [
      {
        "function": "mean",
        "field_name": "response_time",
        "by_field_name": "service_name"
      },
      {
        "function": "count",
        "partition_field_name": "status_code"
      }
    ],
    "influencers": ["service_name", "host"]
  },
  "data_description": {
    "time_field": "@timestamp"
  },
  "model_plot_config": {
    "enabled": true
  }
}'

# Открытие job
curl -X POST "localhost:9200/_ml/anomaly_detectors/request_rate_anomaly/_open"

# Настройка datafeed
curl -X PUT "localhost:9200/_ml/datafeeds/datafeed-request_rate" -H 'Content-Type: application/json' -d'
{
  "job_id": "request_rate_anomaly",
  "indices": ["logs-*"],
  "query": {
    "match_all": {}
  }
}'

# Запуск datafeed
curl -X POST "localhost:9200/_ml/datafeeds/datafeed-request_rate/_start"
```

### Detector Functions

| Function | Описание |
|----------|----------|
| count | Количество событий |
| high_count / low_count | Аномально высокое/низкое количество |
| mean | Среднее значение |
| high_mean / low_mean | Аномально высокое/низкое среднее |
| sum | Сумма |
| median | Медиана |
| min / max | Минимум / максимум |
| distinct_count | Количество уникальных значений |
| rare | Редкие значения |
| freq_rare | Редко встречающиеся паттерны |

### Forecasting

```bash
# Прогнозирование
curl -X POST "localhost:9200/_ml/anomaly_detectors/request_rate_anomaly/_forecast" -H 'Content-Type: application/json' -d'
{
  "duration": "7d"
}'

# Получение результатов прогноза
curl -X GET "localhost:9200/_ml/anomaly_detectors/request_rate_anomaly/results/model_snapshots"
```

### Data Frame Analytics

```bash
# Регрессия
curl -X PUT "localhost:9200/_ml/data_frame/analytics/house_price_prediction" -H 'Content-Type: application/json' -d'
{
  "source": {
    "index": "houses"
  },
  "dest": {
    "index": "house_price_predictions"
  },
  "analysis": {
    "regression": {
      "dependent_variable": "price",
      "training_percent": 80,
      "num_top_feature_importance_values": 10
    }
  }
}'

# Классификация
curl -X PUT "localhost:9200/_ml/data_frame/analytics/fraud_detection" -H 'Content-Type: application/json' -d'
{
  "source": {
    "index": "transactions"
  },
  "dest": {
    "index": "fraud_predictions"
  },
  "analysis": {
    "classification": {
      "dependent_variable": "is_fraud",
      "training_percent": 80,
      "num_top_classes": 2
    }
  }
}'

# Запуск analytics job
curl -X POST "localhost:9200/_ml/data_frame/analytics/house_price_prediction/_start"
```

## Alerting (Watcher)

### Создание watch

```bash
curl -X PUT "localhost:9200/_watcher/watch/high_error_rate" -H 'Content-Type: application/json' -d'
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                { "range": { "@timestamp": { "gte": "now-5m" } } },
                { "term": { "level": "error" } }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total.value": {
        "gt": 100
      }
    }
  },
  "actions": {
    "email_admin": {
      "email": {
        "to": ["admin@example.com"],
        "subject": "High Error Rate Alert",
        "body": {
          "text": "Detected {{ctx.payload.hits.total.value}} errors in the last 5 minutes"
        }
      }
    },
    "slack_notification": {
      "slack": {
        "message": {
          "to": ["#alerts"],
          "text": "High error rate detected: {{ctx.payload.hits.total.value}} errors"
        }
      }
    },
    "webhook": {
      "webhook": {
        "method": "POST",
        "url": "https://api.example.com/alerts",
        "body": "{\"errors\": {{ctx.payload.hits.total.value}}}"
      }
    }
  }
}'
```

### Условия (Conditions)

```json
{
  "condition": {
    "script": {
      "source": "ctx.payload.hits.total.value > params.threshold && ctx.payload.aggregations.avg_response.value > params.max_response",
      "params": {
        "threshold": 100,
        "max_response": 500
      }
    }
  }
}
```

### Throttling

```json
{
  "throttle_period": "15m",
  "actions": {
    "notify": {
      "throttle_period": "1h",
      "email": { ... }
    }
  }
}
```

### Управление watches

```bash
# Активация/деактивация
curl -X PUT "localhost:9200/_watcher/watch/high_error_rate/_activate"
curl -X PUT "localhost:9200/_watcher/watch/high_error_rate/_deactivate"

# Принудительное выполнение
curl -X POST "localhost:9200/_watcher/watch/high_error_rate/_execute"

# История выполнения
curl -X GET "localhost:9200/.watcher-history-*/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "term": { "watch_id": "high_error_rate" }
  },
  "sort": [{ "trigger_event.triggered_time": "desc" }],
  "size": 10
}'
```

## Cross-Cluster Search

### Настройка remote clusters

```bash
# Добавление remote cluster
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_one": {
          "seeds": ["remote-node-1:9300", "remote-node-2:9300"],
          "transport.compress": true,
          "skip_unavailable": true
        },
        "cluster_two": {
          "seeds": ["remote2-node-1:9300"],
          "transport.ping_schedule": "30s"
        }
      }
    }
  }
}'

# Проверка подключения
curl -X GET "localhost:9200/_remote/info?pretty"
```

### Cross-cluster search запросы

```bash
# Поиск по remote cluster
curl -X GET "localhost:9200/cluster_one:products/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": { "name": "laptop" }
  }
}'

# Поиск по нескольким кластерам
curl -X GET "localhost:9200/products,cluster_one:products,cluster_two:products/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": { "name": "laptop" }
  }
}'

# Wildcard patterns
curl -X GET "localhost:9200/*:logs-*/_search"
```

### Cross-Cluster Replication (CCR)

```bash
# Настройка follower index
curl -X PUT "localhost:9200/products-replica/_ccr/follow" -H 'Content-Type: application/json' -d'
{
  "remote_cluster": "cluster_one",
  "leader_index": "products",
  "settings": {
    "index.number_of_replicas": 0
  }
}'

# Auto-follow pattern
curl -X PUT "localhost:9200/_ccr/auto_follow/logs_pattern" -H 'Content-Type: application/json' -d'
{
  "remote_cluster": "cluster_one",
  "leader_index_patterns": ["logs-*"],
  "follow_index_pattern": "{{leader_index}}-replica"
}'

# Проверка статуса
curl -X GET "localhost:9200/products-replica/_ccr/stats?pretty"
```

## Transforms

Transforms позволяют создавать pivot-таблицы и агрегированные представления данных.

### Pivot Transform

```bash
curl -X PUT "localhost:9200/_transform/sales_by_category" -H 'Content-Type: application/json' -d'
{
  "source": {
    "index": ["orders-*"],
    "query": {
      "range": {
        "order_date": {
          "gte": "now-30d"
        }
      }
    }
  },
  "dest": {
    "index": "sales_summary"
  },
  "pivot": {
    "group_by": {
      "category": {
        "terms": {
          "field": "category.keyword"
        }
      },
      "date": {
        "date_histogram": {
          "field": "order_date",
          "calendar_interval": "1d"
        }
      }
    },
    "aggregations": {
      "total_sales": {
        "sum": {
          "field": "amount"
        }
      },
      "order_count": {
        "value_count": {
          "field": "order_id"
        }
      },
      "avg_order_value": {
        "avg": {
          "field": "amount"
        }
      }
    }
  },
  "frequency": "1h",
  "sync": {
    "time": {
      "field": "order_date",
      "delay": "60s"
    }
  }
}'

# Запуск transform
curl -X POST "localhost:9200/_transform/sales_by_category/_start"

# Статус
curl -X GET "localhost:9200/_transform/sales_by_category/_stats?pretty"
```

### Latest Transform

```bash
curl -X PUT "localhost:9200/_transform/latest_status" -H 'Content-Type: application/json' -d'
{
  "source": {
    "index": "device_events"
  },
  "dest": {
    "index": "device_latest_status"
  },
  "latest": {
    "unique_key": ["device_id"],
    "sort": "timestamp"
  }
}'
```

## Searchable Snapshots

```bash
# Монтирование снапшота как searchable
curl -X POST "localhost:9200/_snapshot/my_backup/snapshot_1/_mount" -H 'Content-Type: application/json' -d'
{
  "index": "logs-2023",
  "renamed_index": "logs-2023-searchable",
  "index_settings": {
    "index.number_of_replicas": 0
  }
}'

# Частичное монтирование (frozen tier)
curl -X POST "localhost:9200/_snapshot/my_backup/snapshot_1/_mount?storage=shared_cache" -H 'Content-Type: application/json' -d'
{
  "index": "logs-2023"
}'
```

## Runtime Fields

```bash
# Runtime field в маппинге
curl -X PUT "localhost:9200/products/_mapping" -H 'Content-Type: application/json' -d'
{
  "runtime": {
    "price_with_tax": {
      "type": "double",
      "script": {
        "source": "emit(doc[\"price\"].value * 1.2)"
      }
    },
    "day_of_week": {
      "type": "keyword",
      "script": {
        "source": "emit(doc[\"@timestamp\"].value.dayOfWeekEnum.getDisplayName(TextStyle.FULL, Locale.ROOT))"
      }
    }
  }
}'

# Runtime field в запросе
curl -X GET "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "runtime_mappings": {
    "discount_price": {
      "type": "double",
      "script": {
        "source": "emit(doc[\"price\"].value * 0.9)"
      }
    }
  },
  "query": {
    "range": {
      "discount_price": {
        "lte": 100
      }
    }
  },
  "fields": ["name", "price", "discount_price"]
}'
```

## Async Search

```bash
# Запуск async search
curl -X POST "localhost:9200/logs-*/_async_search" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "by_service": {
      "terms": {
        "field": "service.keyword",
        "size": 1000
      },
      "aggs": {
        "stats": {
          "extended_stats": {
            "field": "response_time"
          }
        }
      }
    }
  }
}'

# Ответ содержит ID
{
  "id": "FmRldE8zREVEUzA2ZVpUeGs2ejJFUFEaBkZ1VXlOTW...",
  "is_partial": true,
  "is_running": true
}

# Проверка статуса
curl -X GET "localhost:9200/_async_search/FmRldE8zREVEUzA2ZVpUeGs2ejJFUFEaBkZ1VXlOTW..."

# Получение результатов с ожиданием
curl -X GET "localhost:9200/_async_search/FmRldE8zREVEUzA2ZVpUeGs2ejJFUFEaBkZ1VXlOTW...?wait_for_completion_timeout=30s"

# Удаление async search
curl -X DELETE "localhost:9200/_async_search/FmRldE8zREVEUzA2ZVpUeGs2ejJFUFEaBkZ1VXlOTW..."
```

## EQL (Event Query Language)

```bash
# Поиск последовательности событий
curl -X GET "localhost:9200/security-events/_eql/search" -H 'Content-Type: application/json' -d'
{
  "query": """
    sequence by user.name with maxspan=1h
      [authentication where event.outcome == "failure"] by source.ip
      [authentication where event.outcome == "failure"] by source.ip
      [authentication where event.outcome == "failure"] by source.ip
      [authentication where event.outcome == "success"] by source.ip
  """
}'

# Простой EQL запрос
curl -X GET "localhost:9200/logs/_eql/search" -H 'Content-Type: application/json' -d'
{
  "query": "process where process.name == \"cmd.exe\" and process.args : \"*powershell*\""
}'
```

## Vector Search (kNN)

```bash
# Создание индекса с dense_vector
curl -X PUT "localhost:9200/image-embeddings" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "image_id": { "type": "keyword" },
      "image_embedding": {
        "type": "dense_vector",
        "dims": 512,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}'

# kNN поиск
curl -X GET "localhost:9200/image-embeddings/_search" -H 'Content-Type: application/json' -d'
{
  "knn": {
    "field": "image_embedding",
    "query_vector": [0.1, 0.2, 0.3, ...],
    "k": 10,
    "num_candidates": 100
  },
  "_source": ["image_id"]
}'

# Hybrid search (kNN + text)
curl -X GET "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "description": "comfortable shoes"
    }
  },
  "knn": {
    "field": "description_embedding",
    "query_vector": [0.1, 0.2, ...],
    "k": 50,
    "num_candidates": 100,
    "boost": 0.5
  },
  "size": 10
}'
```

## Best Practices

### Использование advanced features

```
✅ ML jobs — мониторинг за аномалиями в production
✅ Watcher — alerting для критических метрик
✅ Transforms — предагрегация для дашбордов
✅ CCR — disaster recovery и geo-distribution
✅ Searchable snapshots — экономия на cold storage
✅ Runtime fields — гибкость без переиндексации
```

### Рекомендации

```
✅ Используйте ML для обнаружения аномалий в логах и метриках
✅ Настраивайте throttling для alerting
✅ Тестируйте cross-cluster поиск на latency
✅ Используйте transforms для тяжёлых агрегаций
✅ Применяйте searchable snapshots для архивных данных
```
