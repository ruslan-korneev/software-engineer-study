# Агрегации в Elasticsearch

## Введение в агрегации

Агрегации — это мощный инструмент аналитики в Elasticsearch, позволяющий группировать, суммировать и вычислять статистики по данным. Они работают параллельно с поисковыми запросами и не влияют на результаты поиска.

## Структура агрегации

```bash
curl -X GET "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "query": {
    "match": { "category": "electronics" }
  },
  "aggs": {
    "my_aggregation_name": {
      "aggregation_type": {
        "aggregation_parameters": "..."
      },
      "aggs": {
        "sub_aggregation": { ... }
      }
    }
  }
}'
```

**Примечание:** `size: 0` — не возвращать документы, только агрегации.

## Bucket Aggregations

Bucket-агрегации группируют документы в "корзины" (buckets) по определённым критериям.

### Terms Aggregation

```json
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": {
        "field": "category.keyword",
        "size": 10,
        "order": { "_count": "desc" },
        "min_doc_count": 1,
        "missing": "N/A"
      }
    }
  }
}
```

**Ответ:**
```json
{
  "aggregations": {
    "categories": {
      "buckets": [
        { "key": "electronics", "doc_count": 1500 },
        { "key": "clothing", "doc_count": 1200 },
        { "key": "books", "doc_count": 800 }
      ]
    }
  }
}
```

### Range Aggregation

```json
{
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "keyed": true,
        "ranges": [
          { "key": "cheap", "to": 50 },
          { "key": "medium", "from": 50, "to": 200 },
          { "key": "expensive", "from": 200 }
        ]
      }
    }
  }
}
```

### Date Range Aggregation

```json
{
  "aggs": {
    "sales_periods": {
      "date_range": {
        "field": "created_at",
        "format": "yyyy-MM-dd",
        "ranges": [
          { "key": "last_week", "from": "now-1w/d", "to": "now/d" },
          { "key": "last_month", "from": "now-1M/d", "to": "now-1w/d" },
          { "key": "older", "to": "now-1M/d" }
        ]
      }
    }
  }
}
```

### Histogram Aggregation

```json
{
  "aggs": {
    "price_histogram": {
      "histogram": {
        "field": "price",
        "interval": 50,
        "min_doc_count": 0,
        "extended_bounds": {
          "min": 0,
          "max": 500
        }
      }
    }
  }
}
```

### Date Histogram Aggregation

```json
{
  "aggs": {
    "sales_over_time": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month",
        "format": "yyyy-MM",
        "time_zone": "Europe/Moscow",
        "min_doc_count": 0,
        "extended_bounds": {
          "min": "2024-01-01",
          "max": "2024-12-31"
        }
      }
    }
  }
}
```

**Доступные интервалы:**
- `calendar_interval`: minute, hour, day, week, month, quarter, year
- `fixed_interval`: 1m, 1h, 1d (фиксированная длительность)

### Filter Aggregation

```json
{
  "aggs": {
    "premium_products": {
      "filter": {
        "range": { "price": { "gte": 1000 } }
      },
      "aggs": {
        "avg_rating": {
          "avg": { "field": "rating" }
        }
      }
    }
  }
}
```

### Filters Aggregation

```json
{
  "aggs": {
    "product_types": {
      "filters": {
        "other_bucket_key": "other",
        "filters": {
          "phones": { "term": { "category": "phones" } },
          "laptops": { "term": { "category": "laptops" } },
          "tablets": { "term": { "category": "tablets" } }
        }
      }
    }
  }
}
```

### Nested Aggregation

```json
{
  "aggs": {
    "comments": {
      "nested": {
        "path": "comments"
      },
      "aggs": {
        "top_commenters": {
          "terms": {
            "field": "comments.author.keyword",
            "size": 10
          }
        },
        "avg_rating": {
          "avg": {
            "field": "comments.rating"
          }
        }
      }
    }
  }
}
```

### Geo Distance Aggregation

```json
{
  "aggs": {
    "distance_from_moscow": {
      "geo_distance": {
        "field": "location",
        "origin": { "lat": 55.75, "lon": 37.62 },
        "unit": "km",
        "ranges": [
          { "key": "nearby", "to": 10 },
          { "key": "medium", "from": 10, "to": 50 },
          { "key": "far", "from": 50 }
        ]
      }
    }
  }
}
```

### Composite Aggregation (для пагинации)

```json
{
  "size": 0,
  "aggs": {
    "products_by_category_brand": {
      "composite": {
        "size": 100,
        "sources": [
          { "category": { "terms": { "field": "category.keyword" } } },
          { "brand": { "terms": { "field": "brand.keyword" } } }
        ],
        "after": { "category": "electronics", "brand": "Apple" }
      }
    }
  }
}
```

## Metric Aggregations

Metric-агрегации вычисляют числовые значения на основе полей документов.

### Single-value Metrics

```json
{
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "max_price": { "max": { "field": "price" } },
    "min_price": { "min": { "field": "price" } },
    "sum_quantity": { "sum": { "field": "quantity" } },
    "total_products": { "value_count": { "field": "id" } },
    "unique_brands": { "cardinality": { "field": "brand.keyword" } }
  }
}
```

### Stats Aggregation

```json
{
  "aggs": {
    "price_stats": {
      "stats": {
        "field": "price"
      }
    }
  }
}
```

**Ответ:**
```json
{
  "aggregations": {
    "price_stats": {
      "count": 1000,
      "min": 9.99,
      "max": 2999.99,
      "avg": 299.50,
      "sum": 299500.00
    }
  }
}
```

### Extended Stats Aggregation

```json
{
  "aggs": {
    "price_extended_stats": {
      "extended_stats": {
        "field": "price"
      }
    }
  }
}
```

**Дополнительно к stats:**
- `sum_of_squares`
- `variance`
- `std_deviation`
- `std_deviation_bounds`

### Percentiles Aggregation

```json
{
  "aggs": {
    "price_percentiles": {
      "percentiles": {
        "field": "price",
        "percents": [25, 50, 75, 90, 95, 99]
      }
    }
  }
}
```

### Percentile Ranks Aggregation

```json
{
  "aggs": {
    "price_ranks": {
      "percentile_ranks": {
        "field": "price",
        "values": [100, 500, 1000]
      }
    }
  }
}
```

**Показывает процент документов со значением меньше указанного.**

### Top Hits Aggregation

```json
{
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category.keyword"
      },
      "aggs": {
        "top_products": {
          "top_hits": {
            "size": 3,
            "sort": [{ "rating": "desc" }],
            "_source": ["name", "price", "rating"]
          }
        }
      }
    }
  }
}
```

### Weighted Avg Aggregation

```json
{
  "aggs": {
    "weighted_rating": {
      "weighted_avg": {
        "value": {
          "field": "rating"
        },
        "weight": {
          "field": "reviews_count"
        }
      }
    }
  }
}
```

### Scripted Metric Aggregation

```json
{
  "aggs": {
    "profit": {
      "scripted_metric": {
        "init_script": "state.transactions = []",
        "map_script": "state.transactions.add(doc['price'].value - doc['cost'].value)",
        "combine_script": "double profit = 0; for (t in state.transactions) { profit += t } return profit",
        "reduce_script": "double profit = 0; for (a in states) { profit += a } return profit"
      }
    }
  }
}
```

## Pipeline Aggregations

Pipeline-агрегации работают с результатами других агрегаций, а не напрямую с документами.

### Derivative (производная)

```json
{
  "aggs": {
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "total_sales": {
          "sum": { "field": "amount" }
        },
        "sales_growth": {
          "derivative": {
            "buckets_path": "total_sales"
          }
        }
      }
    }
  }
}
```

### Cumulative Sum

```json
{
  "aggs": {
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "monthly_sales": {
          "sum": { "field": "amount" }
        },
        "cumulative_sales": {
          "cumulative_sum": {
            "buckets_path": "monthly_sales"
          }
        }
      }
    }
  }
}
```

### Moving Average

```json
{
  "aggs": {
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "monthly_sales": {
          "sum": { "field": "amount" }
        },
        "moving_avg_sales": {
          "moving_avg": {
            "buckets_path": "monthly_sales",
            "window": 3,
            "model": "simple"
          }
        }
      }
    }
  }
}
```

### Bucket Script

```json
{
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category.keyword"
      },
      "aggs": {
        "total_revenue": {
          "sum": { "field": "price" }
        },
        "total_cost": {
          "sum": { "field": "cost" }
        },
        "profit_margin": {
          "bucket_script": {
            "buckets_path": {
              "revenue": "total_revenue",
              "cost": "total_cost"
            },
            "script": "(params.revenue - params.cost) / params.revenue * 100"
          }
        }
      }
    }
  }
}
```

### Bucket Sort

```json
{
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category.keyword",
        "size": 100
      },
      "aggs": {
        "total_sales": {
          "sum": { "field": "amount" }
        },
        "top_categories": {
          "bucket_sort": {
            "sort": [
              { "total_sales": { "order": "desc" } }
            ],
            "from": 0,
            "size": 5
          }
        }
      }
    }
  }
}
```

### Bucket Selector

```json
{
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category.keyword"
      },
      "aggs": {
        "avg_rating": {
          "avg": { "field": "rating" }
        },
        "high_rated_only": {
          "bucket_selector": {
            "buckets_path": {
              "avgRating": "avg_rating"
            },
            "script": "params.avgRating >= 4.5"
          }
        }
      }
    }
  }
}
```

## Вложенные агрегации

```json
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": {
        "field": "category.keyword",
        "size": 10
      },
      "aggs": {
        "brands": {
          "terms": {
            "field": "brand.keyword",
            "size": 5
          },
          "aggs": {
            "avg_price": {
              "avg": { "field": "price" }
            },
            "price_ranges": {
              "range": {
                "field": "price",
                "ranges": [
                  { "to": 100 },
                  { "from": 100, "to": 500 },
                  { "from": 500 }
                ]
              }
            }
          }
        },
        "category_stats": {
          "stats": { "field": "price" }
        }
      }
    }
  }
}
```

## Global Aggregation

```json
{
  "query": {
    "term": { "category": "electronics" }
  },
  "aggs": {
    "filtered_stats": {
      "stats": { "field": "price" }
    },
    "all_products": {
      "global": {},
      "aggs": {
        "global_stats": {
          "stats": { "field": "price" }
        }
      }
    }
  }
}
```

## Примеры практического применения

### Фасетный поиск (E-commerce)

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "laptop" } }
      ],
      "filter": [
        { "term": { "in_stock": true } }
      ]
    }
  },
  "aggs": {
    "brands": {
      "terms": { "field": "brand.keyword", "size": 20 }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "key": "under_500", "to": 500 },
          { "key": "500_to_1000", "from": 500, "to": 1000 },
          { "key": "1000_to_2000", "from": 1000, "to": 2000 },
          { "key": "over_2000", "from": 2000 }
        ]
      }
    },
    "screen_sizes": {
      "terms": { "field": "screen_size.keyword" }
    },
    "avg_rating": {
      "avg": { "field": "rating" }
    }
  }
}
```

### Аналитика продаж

```json
{
  "size": 0,
  "aggs": {
    "monthly_sales": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "month"
      },
      "aggs": {
        "revenue": {
          "sum": { "field": "total_amount" }
        },
        "orders_count": {
          "value_count": { "field": "order_id" }
        },
        "avg_order_value": {
          "avg": { "field": "total_amount" }
        },
        "revenue_growth": {
          "derivative": {
            "buckets_path": "revenue"
          }
        }
      }
    },
    "top_products": {
      "terms": {
        "field": "product_name.keyword",
        "size": 10,
        "order": { "revenue": "desc" }
      },
      "aggs": {
        "revenue": {
          "sum": { "field": "total_amount" }
        }
      }
    }
  }
}
```

## Best Practices

### Оптимизация агрегаций

```
✅ Используйте size: 0 если не нужны документы
✅ Используйте keyword поля для terms aggregation
✅ Ограничивайте размер buckets (size параметр)
✅ Используйте shard_size для точности terms aggregation
✅ Кэшируйте результаты агрегаций
✅ Используйте composite aggregation для пагинации
```

### Типичные ошибки

```
❌ Агрегации по text полям (нужен keyword или fielddata)
❌ Слишком много вложенных агрегаций (глубина > 3)
❌ Неограниченный size в terms aggregation
❌ Игнорирование shard_size для распределённых агрегаций
```

### Настройка точности

```json
{
  "aggs": {
    "categories": {
      "terms": {
        "field": "category.keyword",
        "size": 10,
        "shard_size": 50,
        "show_term_doc_count_error": true
      }
    }
  }
}
```
