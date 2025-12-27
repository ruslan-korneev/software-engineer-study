# Анализ текста в Elasticsearch

## Введение в Text Analysis

Text Analysis — это процесс преобразования текста в токены (термины), которые затем индексируются для полнотекстового поиска. Анализ применяется как при индексации документов, так и при выполнении поисковых запросов.

## Процесс анализа

```
Исходный текст: "The QUICK Brown Foxes jumped over the lazy dog's bone."
                            |
                            v
+---------------------------------------------------------------+
|                    Character Filters                           |
|  (Преобразование символов перед токенизацией)                 |
|  Пример: HTML strip, pattern replace                           |
+---------------------------------------------------------------+
                            |
                            v
+---------------------------------------------------------------+
|                      Tokenizer                                 |
|  (Разбиение текста на токены)                                 |
|  "The" "QUICK" "Brown" "Foxes" "jumped" "over" ...            |
+---------------------------------------------------------------+
                            |
                            v
+---------------------------------------------------------------+
|                    Token Filters                               |
|  (Обработка токенов)                                          |
|  lowercase -> "the" "quick" "brown" "foxes" ...               |
|  stemmer -> "the" "quick" "brown" "fox" "jump" ...            |
|  stop -> "quick" "brown" "fox" "jump" ...                     |
+---------------------------------------------------------------+
                            |
                            v
              Индексируемые термины: [quick, brown, fox, jump, ...]
```

## Тестирование анализа

### Analyze API

```bash
# Тестирование стандартного анализатора
curl -X POST "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer": "standard",
  "text": "The QUICK Brown Foxes jumped!"
}'
```

**Результат:**
```json
{
  "tokens": [
    { "token": "the", "start_offset": 0, "end_offset": 3, "position": 0 },
    { "token": "quick", "start_offset": 4, "end_offset": 9, "position": 1 },
    { "token": "brown", "start_offset": 10, "end_offset": 15, "position": 2 },
    { "token": "foxes", "start_offset": 16, "end_offset": 21, "position": 3 },
    { "token": "jumped", "start_offset": 22, "end_offset": 28, "position": 4 }
  ]
}
```

### Тестирование по компонентам

```bash
# Тестирование конкретного токенайзера
curl -X POST "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "tokenizer": "standard",
  "filter": ["lowercase", "porter_stem"],
  "text": "The Quick Foxes jumped"
}'

# Тестирование анализатора индекса
curl -X POST "localhost:9200/my_index/_analyze" -H 'Content-Type: application/json' -d'
{
  "field": "description",
  "text": "Тестовый текст для анализа"
}'
```

## Встроенные анализаторы

### Standard Analyzer (по умолчанию)

```bash
curl -X POST "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer": "standard",
  "text": "The Quick-Brown fox email: fox@example.com"
}'
```

**Результат:** `[the, quick, brown, fox, email, fox, example.com]`

### Simple Analyzer

```json
{
  "analyzer": "simple",
  "text": "The Quick-Brown FOX's 123"
}
```

**Результат:** `[the, quick, brown, fox, s]` (разбивает по не-буквам, lowercase)

### Whitespace Analyzer

```json
{
  "analyzer": "whitespace",
  "text": "The Quick-Brown FOX"
}
```

**Результат:** `[The, Quick-Brown, FOX]` (только по пробелам, без lowercase)

### Keyword Analyzer

```json
{
  "analyzer": "keyword",
  "text": "The Quick Brown Fox"
}
```

**Результат:** `[The Quick Brown Fox]` (весь текст как один токен)

### Language Analyzers

```bash
# Русский анализатор
curl -X POST "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer": "russian",
  "text": "Быстрые коричневые лисы прыгают через ленивых собак"
}'
```

**Результат:** `[быстр, коричнев, лис, прыга, через, ленив, собак]` (стемминг)

```bash
# Английский анализатор
curl -X POST "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer": "english",
  "text": "The quick brown foxes were jumping"
}'
```

**Результат:** `[quick, brown, fox, jump]` (стоп-слова удалены, стемминг)

## Tokenizers

### Standard Tokenizer

```json
{
  "tokenizer": "standard",
  "text": "email@example.com user-name test_123"
}
```

**Результат:** `[email, example.com, user, name, test_123]`

### Letter Tokenizer

```json
{
  "tokenizer": "letter",
  "text": "hello-world test123"
}
```

**Результат:** `[hello, world, test]`

### Whitespace Tokenizer

```json
{
  "tokenizer": "whitespace",
  "text": "hello-world test123"
}
```

**Результат:** `[hello-world, test123]`

### N-gram Tokenizer

```bash
curl -X PUT "localhost:9200/ngram_index" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "analysis": {
      "tokenizer": {
        "my_ngram": {
          "type": "ngram",
          "min_gram": 2,
          "max_gram": 3,
          "token_chars": ["letter", "digit"]
        }
      },
      "analyzer": {
        "ngram_analyzer": {
          "tokenizer": "my_ngram"
        }
      }
    }
  }
}'
```

**"Quick"** -> `[Qu, Qui, ui, uic, ic, ick, ck]`

### Edge N-gram Tokenizer

```json
{
  "tokenizer": {
    "type": "edge_ngram",
    "min_gram": 1,
    "max_gram": 10,
    "token_chars": ["letter"]
  }
}
```

**"Quick"** -> `[Q, Qu, Qui, Quic, Quick]` (для автодополнения)

### Pattern Tokenizer

```json
{
  "tokenizer": {
    "type": "pattern",
    "pattern": "[\\W_]+",
    "lowercase": true
  }
}
```

### Path Hierarchy Tokenizer

```json
{
  "tokenizer": "path_hierarchy",
  "text": "/users/john/documents/report.pdf"
}
```

**Результат:** `[/users, /users/john, /users/john/documents, /users/john/documents/report.pdf]`

## Character Filters

### HTML Strip

```bash
curl -X POST "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "char_filter": ["html_strip"],
  "tokenizer": "standard",
  "text": "<p>Hello <b>World</b>!</p>"
}'
```

**Результат:** `[hello, world]`

### Mapping Character Filter

```json
{
  "settings": {
    "analysis": {
      "char_filter": {
        "emoticons": {
          "type": "mapping",
          "mappings": [
            ":) => happy",
            ":( => sad",
            ":D => excited"
          ]
        }
      },
      "analyzer": {
        "emoticon_analyzer": {
          "char_filter": ["emoticons"],
          "tokenizer": "standard"
        }
      }
    }
  }
}
```

### Pattern Replace

```json
{
  "char_filter": {
    "phone_filter": {
      "type": "pattern_replace",
      "pattern": "(\\d{3})-(\\d{3})-(\\d{4})",
      "replacement": "$1$2$3"
    }
  }
}
```

## Token Filters

### Lowercase / Uppercase

```json
{
  "filter": ["lowercase"],
  "text": "HELLO World"
}
```

### Stop Words

```json
{
  "settings": {
    "analysis": {
      "filter": {
        "russian_stop": {
          "type": "stop",
          "stopwords": "_russian_"
        },
        "custom_stop": {
          "type": "stop",
          "stopwords": ["the", "a", "an", "и", "в", "на"]
        }
      }
    }
  }
}
```

### Stemmer

```json
{
  "settings": {
    "analysis": {
      "filter": {
        "russian_stemmer": {
          "type": "stemmer",
          "language": "russian"
        },
        "english_stemmer": {
          "type": "stemmer",
          "language": "porter2"
        }
      }
    }
  }
}
```

### Synonym Filter

```json
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonyms": {
          "type": "synonym",
          "synonyms": [
            "quick, fast, speedy",
            "big, large, huge => large",
            "laptop => notebook, portable computer"
          ]
        },
        "synonym_graph": {
          "type": "synonym_graph",
          "synonyms_path": "analysis/synonyms.txt"
        }
      }
    }
  }
}
```

**Файл synonyms.txt:**
```
# Формат: слово1, слово2, слово3
# или: слово1, слово2 => замена
iphone, айфон
macbook, макбук => ноутбук apple
```

### ASCII Folding

```json
{
  "filter": "asciifolding",
  "text": "cafe resume naive"
}
```

**Результат:** `[cafe, resume, naive]`

### Word Delimiter Graph

```json
{
  "filter": {
    "my_word_delimiter": {
      "type": "word_delimiter_graph",
      "split_on_case_change": true,
      "split_on_numerics": true,
      "preserve_original": true
    }
  }
}
```

**"PowerShot2000"** -> `[PowerShot2000, Power, Shot, 2000]`

### Edge N-gram Filter

```json
{
  "filter": {
    "autocomplete_filter": {
      "type": "edge_ngram",
      "min_gram": 1,
      "max_gram": 20
    }
  }
}
```

### Phonetic Filter

```json
{
  "filter": {
    "phonetic_filter": {
      "type": "phonetic",
      "encoder": "metaphone",
      "replace": false
    }
  }
}
```

## Создание кастомного анализатора

### Пример: Анализатор для русского текста

```bash
curl -X PUT "localhost:9200/russian_index" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "analysis": {
      "char_filter": {
        "yo_replace": {
          "type": "mapping",
          "mappings": ["ё => е"]
        }
      },
      "filter": {
        "russian_stop": {
          "type": "stop",
          "stopwords": "_russian_"
        },
        "russian_stemmer": {
          "type": "stemmer",
          "language": "russian"
        },
        "custom_synonyms": {
          "type": "synonym",
          "synonyms": [
            "телефон, смартфон, мобильный"
          ]
        }
      },
      "analyzer": {
        "russian_custom": {
          "type": "custom",
          "char_filter": ["yo_replace"],
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "russian_stop",
            "russian_stemmer",
            "custom_synonyms"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "analyzer": "russian_custom"
      }
    }
  }
}'
```

### Пример: Автодополнение (Autocomplete)

```bash
curl -X PUT "localhost:9200/autocomplete_index" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "analysis": {
      "filter": {
        "autocomplete_filter": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 20
        }
      },
      "analyzer": {
        "autocomplete_index": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "autocomplete_filter"
          ]
        },
        "autocomplete_search": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "autocomplete_index",
        "search_analyzer": "autocomplete_search"
      }
    }
  }
}'
```

**Использование:**
```json
{
  "query": {
    "match": {
      "name": {
        "query": "lap",
        "operator": "and"
      }
    }
  }
}
```

### Пример: Поиск с опечатками

```bash
curl -X PUT "localhost:9200/fuzzy_index" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "analysis": {
      "filter": {
        "trigram_filter": {
          "type": "ngram",
          "min_gram": 3,
          "max_gram": 3
        }
      },
      "analyzer": {
        "trigram_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "trigram_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "trigram": {
            "type": "text",
            "analyzer": "trigram_analyzer"
          }
        }
      }
    }
  }
}'
```

## Search-time vs Index-time Analysis

### Разные анализаторы

```json
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "analyzer": "english",
        "search_analyzer": "english_exact"
      }
    }
  }
}
```

### Когда использовать разные анализаторы

| Сценарий | Index Analyzer | Search Analyzer |
|----------|---------------|-----------------|
| Автодополнение | edge_ngram | standard |
| Синонимы (расширение) | с синонимами | без синонимов |
| Стемминг | со стеммингом | без стемминга (для точного поиска) |

## Normalizers (для keyword полей)

```bash
curl -X PUT "localhost:9200/normalized_index" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "analysis": {
      "normalizer": {
        "lowercase_normalizer": {
          "type": "custom",
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "email": {
        "type": "keyword",
        "normalizer": "lowercase_normalizer"
      }
    }
  }
}'
```

## Best Practices

### Выбор анализатора

```
✅ Используйте language analyzer для полнотекстового поиска
✅ Используйте edge_ngram для автодополнения
✅ Тестируйте анализаторы с Analyze API перед использованием
✅ Храните синонимы в файлах для удобства обновления
✅ Используйте search_analyzer отдельно от index analyzer
```

### Типичные ошибки

```
❌ Использование стандартного анализатора для русского текста
❌ Слишком длинные n-grams (увеличивает размер индекса)
❌ Синонимы только на search-time (работает, но менее эффективно)
❌ Забыть про normalization для keyword полей
❌ Игнорирование stop words для разных языков
```

### Оптимизация

```json
{
  "settings": {
    "index": {
      "max_ngram_diff": 10,
      "similarity": {
        "default": {
          "type": "BM25",
          "b": 0.75,
          "k1": 1.2
        }
      }
    }
  }
}
```

## Отладка анализа

```bash
# Explain scoring с анализом
curl -X GET "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "explain": true,
  "query": {
    "match": {
      "name": "laptop"
    }
  }
}'

# Проверка термов в индексе
curl -X GET "localhost:9200/products/_termvectors/1" -H 'Content-Type: application/json' -d'
{
  "fields": ["name"],
  "term_statistics": true,
  "field_statistics": true
}'
```
