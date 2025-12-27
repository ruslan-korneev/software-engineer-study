# Релевантность и тюнинг поиска в Elasticsearch

## Введение в Scoring

Elasticsearch использует алгоритм BM25 (Okapi BM25) для расчёта релевантности документов. Score определяет, насколько хорошо документ соответствует запросу.

## Алгоритм BM25

### Формула BM25

```
score(D, Q) = Σ IDF(qi) × (f(qi, D) × (k1 + 1)) / (f(qi, D) + k1 × (1 - b + b × |D|/avgdl))

Где:
- D — документ
- Q — запрос
- qi — термин запроса
- f(qi, D) — частота термина в документе
- |D| — длина документа
- avgdl — средняя длина документов
- k1 — параметр насыщения термина (default: 1.2)
- b — параметр нормализации длины (default: 0.75)
```

### Настройка параметров BM25

```bash
curl -X PUT "localhost:9200/products" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "index": {
      "similarity": {
        "custom_bm25": {
          "type": "BM25",
          "k1": 1.2,
          "b": 0.75
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "similarity": "custom_bm25"
      }
    }
  }
}'
```

### Параметры BM25

| Параметр | Диапазон | Описание | Значение по умолчанию |
|----------|----------|----------|----------------------|
| **k1** | 0-3 | Насыщение частоты термина | 1.2 |
| **b** | 0-1 | Влияние длины документа | 0.75 |

- `k1 = 0`: частота термина не учитывается
- `k1 > 2`: высокая частота сильно влияет
- `b = 0`: длина документа не учитывается
- `b = 1`: полная нормализация по длине

## Explain API

### Анализ scoring

```bash
curl -X GET "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "explain": true,
  "query": {
    "match": {
      "name": "gaming laptop"
    }
  }
}'
```

**Ответ с explain:**
```json
{
  "_explanation": {
    "value": 5.2,
    "description": "sum of:",
    "details": [
      {
        "value": 2.8,
        "description": "weight(name:gaming in 0) [PerFieldSimilarity]",
        "details": [
          {
            "value": 1.5,
            "description": "idf, computed as log(1 + (N - n + 0.5) / (n + 0.5))"
          },
          {
            "value": 1.86,
            "description": "tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl))"
          }
        ]
      },
      {
        "value": 2.4,
        "description": "weight(name:laptop in 0) [PerFieldSimilarity]"
      }
    ]
  }
}
```

### Explain для конкретного документа

```bash
curl -X GET "localhost:9200/products/_explain/123" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "name": "gaming laptop"
    }
  }
}'
```

## Boosting

### Field Boosting (в запросе)

```json
{
  "query": {
    "multi_match": {
      "query": "gaming laptop",
      "fields": ["name^3", "description^2", "category^1"]
    }
  }
}
```

### Query Boosting

```json
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "name": {
              "query": "gaming laptop",
              "boost": 3
            }
          }
        },
        {
          "match": {
            "description": {
              "query": "gaming laptop",
              "boost": 1
            }
          }
        }
      ]
    }
  }
}
```

### Negative Boosting

```json
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "name": "laptop"
        }
      },
      "negative": {
        "term": {
          "condition": "refurbished"
        }
      },
      "negative_boost": 0.5
    }
  }
}
```

## Function Score Query

Function Score — мощный инструмент для кастомизации scoring.

### Базовая структура

```json
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "functions": [
        { "function_type": { ... } }
      ],
      "score_mode": "multiply",
      "boost_mode": "multiply",
      "max_boost": 10
    }
  }
}
```

### Score Mode и Boost Mode

| score_mode | Описание |
|------------|----------|
| multiply | Перемножение scores функций |
| sum | Сумма |
| avg | Среднее |
| first | Первая подходящая функция |
| max | Максимальное |
| min | Минимальное |

| boost_mode | Описание |
|------------|----------|
| multiply | query_score × function_score |
| replace | Только function_score |
| sum | query_score + function_score |
| avg | Среднее |
| max | Максимальное |
| min | Минимальное |

### Field Value Factor

```json
{
  "query": {
    "function_score": {
      "query": {
        "match": { "name": "laptop" }
      },
      "functions": [
        {
          "field_value_factor": {
            "field": "popularity",
            "factor": 1.2,
            "modifier": "sqrt",
            "missing": 1
          }
        }
      ],
      "boost_mode": "multiply"
    }
  }
}
```

**Модификаторы:**
- `none`: значение поля
- `log`: log(значение)
- `log1p`: log(1 + значение)
- `log2p`: log(2 + значение)
- `ln`: ln(значение)
- `ln1p`: ln(1 + значение)
- `ln2p`: ln(2 + значение)
- `square`: значение²
- `sqrt`: √значение
- `reciprocal`: 1/значение

### Decay Functions

Decay функции уменьшают score в зависимости от расстояния от указанной точки.

```json
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "functions": [
        {
          "gauss": {
            "location": {
              "origin": { "lat": 55.75, "lon": 37.62 },
              "scale": "5km",
              "offset": "1km",
              "decay": 0.5
            }
          }
        },
        {
          "exp": {
            "price": {
              "origin": 100,
              "scale": 50,
              "decay": 0.5
            }
          }
        },
        {
          "linear": {
            "date": {
              "origin": "now",
              "scale": "10d",
              "decay": 0.5
            }
          }
        }
      ],
      "score_mode": "multiply"
    }
  }
}
```

**Типы decay:**
- `gauss`: Гауссово убывание
- `exp`: Экспоненциальное
- `linear`: Линейное

### Script Score

```json
{
  "query": {
    "function_score": {
      "query": { "match": { "name": "laptop" } },
      "functions": [
        {
          "script_score": {
            "script": {
              "source": "_score * doc['popularity'].value * Math.log(2 + doc['reviews_count'].value)"
            }
          }
        }
      ]
    }
  }
}
```

### Weight Function

```json
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "functions": [
        {
          "filter": { "term": { "featured": true } },
          "weight": 2
        },
        {
          "filter": { "range": { "rating": { "gte": 4.5 } } },
          "weight": 1.5
        },
        {
          "filter": { "term": { "in_stock": true } },
          "weight": 1.2
        }
      ],
      "score_mode": "multiply",
      "boost_mode": "multiply"
    }
  }
}
```

### Random Score

```json
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "functions": [
        {
          "random_score": {
            "seed": 12345,
            "field": "_seq_no"
          }
        }
      ]
    }
  }
}
```

## Комплексный пример Function Score

```json
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {
              "multi_match": {
                "query": "gaming laptop",
                "fields": ["name^3", "description"]
              }
            }
          ],
          "filter": [
            { "term": { "in_stock": true } }
          ]
        }
      },
      "functions": [
        {
          "filter": { "term": { "featured": true } },
          "weight": 3
        },
        {
          "field_value_factor": {
            "field": "rating",
            "factor": 1.5,
            "modifier": "sqrt",
            "missing": 3
          }
        },
        {
          "gauss": {
            "created_at": {
              "origin": "now",
              "scale": "30d",
              "decay": 0.5
            }
          }
        },
        {
          "script_score": {
            "script": {
              "source": "Math.log(2 + doc['sales_count'].value)"
            }
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply",
      "max_boost": 100
    }
  }
}
```

## Rescore Query

Rescore позволяет применить более дорогой запрос к top-N результатам.

```json
{
  "query": {
    "match": {
      "name": {
        "query": "gaming laptop",
        "operator": "or"
      }
    }
  },
  "rescore": {
    "window_size": 100,
    "query": {
      "rescore_query": {
        "match_phrase": {
          "name": {
            "query": "gaming laptop",
            "slop": 2
          }
        }
      },
      "query_weight": 0.7,
      "rescore_query_weight": 1.2
    }
  }
}
```

### Множественный rescore

```json
{
  "query": { "match": { "name": "laptop" } },
  "rescore": [
    {
      "window_size": 100,
      "query": {
        "rescore_query": { "match_phrase": { "name": "gaming laptop" } },
        "query_weight": 0.7,
        "rescore_query_weight": 1.2
      }
    },
    {
      "window_size": 50,
      "query": {
        "rescore_query": {
          "function_score": {
            "script_score": {
              "script": "_score * doc['popularity'].value"
            }
          }
        }
      }
    }
  ]
}
```

## Index Boost

```json
{
  "indices_boost": [
    { "products_new": 1.5 },
    { "products_old": 0.5 }
  ],
  "query": {
    "match": { "name": "laptop" }
  }
}
```

## Практические сценарии

### E-commerce: Релевантность товаров

```json
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {
              "multi_match": {
                "query": "ноутбук для игр",
                "fields": ["name^3", "description^2", "category"],
                "type": "best_fields",
                "fuzziness": "AUTO"
              }
            }
          ],
          "filter": [
            { "term": { "in_stock": true } },
            { "range": { "price": { "gte": 50000, "lte": 150000 } } }
          ]
        }
      },
      "functions": [
        {
          "filter": { "term": { "featured": true } },
          "weight": 2
        },
        {
          "filter": { "range": { "rating": { "gte": 4.5 } } },
          "weight": 1.5
        },
        {
          "field_value_factor": {
            "field": "sales_count",
            "modifier": "log1p",
            "factor": 0.1
          }
        },
        {
          "gauss": {
            "created_at": {
              "origin": "now",
              "scale": "90d",
              "decay": 0.5
            }
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply"
    }
  }
}
```

### Поиск статей с учётом свежести

```json
{
  "query": {
    "function_score": {
      "query": {
        "match": { "content": "elasticsearch" }
      },
      "functions": [
        {
          "exp": {
            "published_at": {
              "origin": "now",
              "scale": "7d",
              "decay": 0.5
            }
          },
          "weight": 2
        },
        {
          "field_value_factor": {
            "field": "views",
            "modifier": "log1p"
          },
          "weight": 0.5
        }
      ],
      "score_mode": "sum"
    }
  }
}
```

## Best Practices

### Тюнинг релевантности

```
✅ Используйте explain для понимания scoring
✅ Начинайте с простых boost значений
✅ Тестируйте изменения на реальных данных
✅ Используйте A/B тестирование
✅ Документируйте логику scoring
✅ Используйте rescore для дорогих операций
```

### Типичные ошибки

```
❌ Слишком высокие boost значения (> 10)
❌ Игнорирование BM25 в пользу только function_score
❌ Сложные скрипты без кэширования
❌ Отсутствие тестирования изменений
```

### Мониторинг качества поиска

```python
# Метрики качества поиска
metrics = {
    "precision_at_k": "Точность в top-K результатах",
    "recall": "Полнота покрытия",
    "ndcg": "Normalized Discounted Cumulative Gain",
    "mrr": "Mean Reciprocal Rank",
    "click_through_rate": "CTR на результаты поиска"
}
```
