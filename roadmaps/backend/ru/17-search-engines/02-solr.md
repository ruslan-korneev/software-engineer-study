# Apache Solr

## Введение

**Apache Solr** — это высокопроизводительная платформа полнотекстового поиска с открытым исходным кодом, построенная на базе библиотеки Apache Lucene. Solr предоставляет распределённую индексацию, репликацию, балансировку нагрузки и централизованное управление конфигурацией.

### История

- **2004** — Solr создан в CNET Networks Йоником Сили
- **2006** — передан в Apache Software Foundation
- **2008** — выпущена версия 1.3 с поддержкой распределённого поиска
- **2012** — Solr 4.0 с SolrCloud для автоматического шардирования
- **2016** — Solr 6.0 с параллельным SQL и потоковыми выражениями
- **2019** — Solr 8.0 с улучшенной автоскейлингом
- **2022** — Solr 9.0 с переходом на Java 11+

### Ключевые особенности

- Полнотекстовый поиск с поддержкой множества языков
- Фасетный поиск и аналитика
- Геопространственный поиск
- Кластеризация документов
- Rich document handling (PDF, Word, Excel)
- RESTful API (JSON, XML)
- Веб-интерфейс администрирования
- Расширяемая плагинная архитектура

---

## Архитектура

### Standalone Mode

В режиме standalone Solr работает как единственный экземпляр:

```
┌─────────────────────────────────┐
│           Solr Instance         │
│  ┌──────────┐  ┌──────────┐    │
│  │  Core 1  │  │  Core 2  │    │
│  │ (index)  │  │ (index)  │    │
│  └──────────┘  └──────────┘    │
└─────────────────────────────────┘
```

**Core** — это отдельный индекс с собственной конфигурацией и схемой.

### SolrCloud

SolrCloud — распределённый режим работы для высокой доступности и масштабируемости:

```
┌─────────────────────────────────────────────────────────────┐
│                       SolrCloud Cluster                      │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │   ZooKeeper     │  │   ZooKeeper     │  (конфигурация   │
│  │   Ensemble      │  │   Node 2        │   и координация) │
│  └────────┬────────┘  └────────┬────────┘                  │
│           │                    │                            │
│  ┌────────┴────────────────────┴────────┐                  │
│  │                                       │                  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  │  Solr    │  │  Solr    │  │  Solr    │              │
│  │  │  Node 1  │  │  Node 2  │  │  Node 3  │              │
│  │  │          │  │          │  │          │              │
│  │  │ Shard1   │  │ Shard2   │  │ Shard1   │              │
│  │  │ (leader) │  │ (leader) │  │ (replica)│              │
│  │  └──────────┘  └──────────┘  └──────────┘              │
│  └───────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

### Основные компоненты

**Collection (Коллекция)** — логический индекс, распределённый по кластеру:

```bash
# Создание коллекции
curl "http://localhost:8983/solr/admin/collections?action=CREATE&name=products&numShards=2&replicationFactor=2"
```

**Shard (Шард)** — горизонтальный раздел коллекции:
- Каждый шард содержит подмножество документов
- Документы распределяются по шардам с помощью хэш-функции от уникального ID

**Replica (Реплика)** — копия шарда для отказоустойчивости:
- **Leader** — принимает запросы на индексацию
- **Replica** — обслуживает запросы на чтение

**ZooKeeper** — координационный сервис:
- Хранит конфигурацию кластера
- Отслеживает состояние нод
- Выбирает лидеров шардов
- Распределяет запросы

### Конфигурация ZooKeeper

```bash
# Загрузка конфигурации в ZooKeeper
bin/solr zk upconfig -n myconfig -d /path/to/configset -z localhost:2181

# Скачивание конфигурации
bin/solr zk downconfig -n myconfig -d /path/to/download -z localhost:2181

# Просмотр содержимого ZooKeeper
bin/solr zk ls /configs -z localhost:2181
```

---

## Схема данных

### Типы схем

**Managed Schema** (рекомендуется) — схема управляется через API:

```xml
<!-- managed-schema -->
<?xml version="1.0" encoding="UTF-8"?>
<schema name="products" version="1.6">
    <uniqueKey>id</uniqueKey>

    <!-- Типы полей -->
    <fieldType name="string" class="solr.StrField" sortMissingLast="true"/>
    <fieldType name="plong" class="solr.LongPointField" docValues="true"/>
    <fieldType name="pfloat" class="solr.FloatPointField" docValues="true"/>
    <fieldType name="pdate" class="solr.DatePointField" docValues="true"/>
    <fieldType name="boolean" class="solr.BoolField" sortMissingLast="true"/>

    <!-- Текстовый тип с анализом -->
    <fieldType name="text_general" class="solr.TextField" positionIncrementGap="100">
        <analyzer type="index">
            <tokenizer class="solr.StandardTokenizerFactory"/>
            <filter class="solr.LowerCaseFilterFactory"/>
        </analyzer>
        <analyzer type="query">
            <tokenizer class="solr.StandardTokenizerFactory"/>
            <filter class="solr.LowerCaseFilterFactory"/>
        </analyzer>
    </fieldType>

    <!-- Определения полей -->
    <field name="id" type="string" indexed="true" stored="true" required="true"/>
    <field name="name" type="text_general" indexed="true" stored="true"/>
    <field name="description" type="text_general" indexed="true" stored="true"/>
    <field name="price" type="pfloat" indexed="true" stored="true"/>
    <field name="category" type="string" indexed="true" stored="true" docValues="true"/>
    <field name="in_stock" type="boolean" indexed="true" stored="true"/>
    <field name="created_at" type="pdate" indexed="true" stored="true"/>

    <!-- Динамические поля -->
    <dynamicField name="*_txt" type="text_general" indexed="true" stored="true"/>
    <dynamicField name="*_i" type="plong" indexed="true" stored="true"/>
    <dynamicField name="*_s" type="string" indexed="true" stored="true"/>

    <!-- Поле для копирования (catch-all) -->
    <field name="_text_" type="text_general" indexed="true" stored="false" multiValued="true"/>
    <copyField source="name" dest="_text_"/>
    <copyField source="description" dest="_text_"/>
</schema>
```

### Schema API

```bash
# Добавление поля через API
curl -X POST "http://localhost:8983/solr/products/schema" \
  -H "Content-Type: application/json" \
  -d '{
    "add-field": {
      "name": "brand",
      "type": "string",
      "indexed": true,
      "stored": true,
      "docValues": true
    }
  }'

# Добавление типа поля
curl -X POST "http://localhost:8983/solr/products/schema" \
  -H "Content-Type: application/json" \
  -d '{
    "add-field-type": {
      "name": "text_ru",
      "class": "solr.TextField",
      "analyzer": {
        "tokenizer": {"class": "solr.StandardTokenizerFactory"},
        "filters": [
          {"class": "solr.LowerCaseFilterFactory"},
          {"class": "solr.SnowballPorterFilterFactory", "language": "Russian"}
        ]
      }
    }
  }'

# Удаление поля
curl -X POST "http://localhost:8983/solr/products/schema" \
  -H "Content-Type: application/json" \
  -d '{"delete-field": {"name": "old_field"}}'
```

### Атрибуты полей

| Атрибут | Описание |
|---------|----------|
| `indexed` | Поле доступно для поиска |
| `stored` | Значение возвращается в результатах |
| `docValues` | Колоночное хранение для сортировки и фасетов |
| `multiValued` | Поле может содержать массив значений |
| `required` | Обязательное поле |
| `omitNorms` | Отключить нормализацию (экономия памяти) |

---

## Индексирование

### POST запросы (JSON)

```bash
# Добавление одного документа
curl -X POST "http://localhost:8983/solr/products/update?commit=true" \
  -H "Content-Type: application/json" \
  -d '{
    "add": {
      "doc": {
        "id": "1",
        "name": "iPhone 15 Pro",
        "description": "Флагманский смартфон Apple с чипом A17 Pro",
        "price": 99990.00,
        "category": "smartphones",
        "in_stock": true,
        "created_at": "2024-01-15T10:30:00Z"
      }
    }
  }'

# Добавление нескольких документов
curl -X POST "http://localhost:8983/solr/products/update?commit=true" \
  -H "Content-Type: application/json" \
  -d '[
    {"id": "2", "name": "Samsung Galaxy S24", "price": 79990, "category": "smartphones"},
    {"id": "3", "name": "MacBook Pro 16", "price": 249990, "category": "laptops"},
    {"id": "4", "name": "iPad Pro", "price": 89990, "category": "tablets"}
  ]'

# Частичное обновление (atomic update)
curl -X POST "http://localhost:8983/solr/products/update?commit=true" \
  -H "Content-Type: application/json" \
  -d '{
    "add": {
      "doc": {
        "id": "1",
        "price": {"set": 94990},
        "tags": {"add": "sale"}
      }
    }
  }'

# Удаление документа
curl -X POST "http://localhost:8983/solr/products/update?commit=true" \
  -H "Content-Type: application/json" \
  -d '{"delete": {"id": "1"}}'

# Удаление по запросу
curl -X POST "http://localhost:8983/solr/products/update?commit=true" \
  -H "Content-Type: application/json" \
  -d '{"delete": {"query": "category:old_category"}}'
```

### Data Import Handler (DIH)

Конфигурация для импорта из базы данных:

```xml
<!-- data-config.xml -->
<dataConfig>
    <dataSource type="JdbcDataSource"
                driver="org.postgresql.Driver"
                url="jdbc:postgresql://localhost:5432/shop"
                user="solr"
                password="password"/>

    <document>
        <entity name="product"
                query="SELECT id, name, description, price, category_id, created_at FROM products"
                deltaQuery="SELECT id FROM products WHERE updated_at > '${dataimporter.last_index_time}'"
                deltaImportQuery="SELECT id, name, description, price, category_id, created_at FROM products WHERE id='${dataimporter.delta.id}'">

            <field column="id" name="id"/>
            <field column="name" name="name"/>
            <field column="description" name="description"/>
            <field column="price" name="price"/>
            <field column="created_at" name="created_at"/>

            <!-- Связанная сущность -->
            <entity name="category"
                    query="SELECT name FROM categories WHERE id='${product.category_id}'">
                <field column="name" name="category"/>
            </entity>
        </entity>
    </document>
</dataConfig>
```

```bash
# Полный импорт
curl "http://localhost:8983/solr/products/dataimport?command=full-import"

# Инкрементальный импорт
curl "http://localhost:8983/solr/products/dataimport?command=delta-import"

# Статус импорта
curl "http://localhost:8983/solr/products/dataimport?command=status"
```

### SolrJ (Java клиент)

```java
import org.apache.solr.client.solrj.SolrClient;
import org.apache.solr.client.solrj.impl.CloudSolrClient;
import org.apache.solr.client.solrj.impl.Http2SolrClient;
import org.apache.solr.common.SolrInputDocument;

// Standalone клиент
SolrClient client = new Http2SolrClient.Builder("http://localhost:8983/solr")
    .build();

// SolrCloud клиент
SolrClient cloudClient = new CloudSolrClient.Builder(
    List.of("zk1:2181", "zk2:2181", "zk3:2181"),
    Optional.empty()
).build();

// Добавление документа
SolrInputDocument doc = new SolrInputDocument();
doc.addField("id", "5");
doc.addField("name", "Google Pixel 8");
doc.addField("price", 69990.0);
doc.addField("category", "smartphones");

client.add("products", doc);
client.commit("products");

// Batch индексирование
List<SolrInputDocument> docs = new ArrayList<>();
for (Product product : products) {
    SolrInputDocument d = new SolrInputDocument();
    d.addField("id", product.getId());
    d.addField("name", product.getName());
    d.addField("price", product.getPrice());
    docs.add(d);
}
client.add("products", docs);
client.commit("products");

// Закрытие клиента
client.close();
```

### Commit стратегии

```xml
<!-- solrconfig.xml -->
<updateHandler class="solr.DirectUpdateHandler2">
    <!-- Auto soft commit каждые 1000 документов или 15 секунд -->
    <autoSoftCommit>
        <maxDocs>1000</maxDocs>
        <maxTime>15000</maxTime>
    </autoSoftCommit>

    <!-- Auto hard commit каждые 10000 документов или 60 секунд -->
    <autoCommit>
        <maxDocs>10000</maxDocs>
        <maxTime>60000</maxTime>
        <openSearcher>false</openSearcher>
    </autoCommit>
</updateHandler>
```

- **Soft commit** — делает документы видимыми для поиска без fsync на диск
- **Hard commit** — записывает данные на диск для durability

---

## Поиск

### Базовые запросы

```bash
# Простой поиск
curl "http://localhost:8983/solr/products/select?q=iphone"

# Поиск по конкретному полю
curl "http://localhost:8983/solr/products/select?q=name:iphone"

# Выбор полей для возврата
curl "http://localhost:8983/solr/products/select?q=*:*&fl=id,name,price"

# Фильтрация
curl "http://localhost:8983/solr/products/select?q=*:*&fq=category:smartphones&fq=price:[50000 TO 100000]"

# Сортировка
curl "http://localhost:8983/solr/products/select?q=*:*&sort=price desc"

# Пагинация
curl "http://localhost:8983/solr/products/select?q=*:*&start=10&rows=20"
```

### Query Parsers

**Standard Query Parser (lucene)**:

```bash
# Булевы операторы
q=iphone AND pro
q=iphone OR samsung
q=iphone NOT mini
q=(iphone OR samsung) AND price:[50000 TO *]

# Фразовый поиск
q="iphone 15 pro"

# Wildcard
q=name:iph*
q=name:ipho?e

# Fuzzy поиск
q=name:iphon~2

# Proximity поиск (слова в пределах 5 слов друг от друга)
q="iphone pro"~5

# Boost
q=name:iphone^2 OR description:iphone

# Range запросы
q=price:[1000 TO 50000]
q=created_at:[2024-01-01T00:00:00Z TO NOW]
```

**DisMax Parser**:

```bash
curl "http://localhost:8983/solr/products/select" \
  --data-urlencode "q=iphone pro" \
  --data-urlencode "defType=dismax" \
  --data-urlencode "qf=name^3 description^1" \
  --data-urlencode "pf=name^5" \
  --data-urlencode "mm=75%"
```

**Extended DisMax (edismax)**:

```bash
curl "http://localhost:8983/solr/products/select" \
  --data-urlencode "q=iphone -mini" \
  --data-urlencode "defType=edismax" \
  --data-urlencode "qf=name^3 description" \
  --data-urlencode "pf=name^5" \
  --data-urlencode "ps=2" \
  --data-urlencode "mm=2<75%"
```

### Фасетный поиск

```bash
# Фасеты по полю
curl "http://localhost:8983/solr/products/select?q=*:*&facet=true&facet.field=category&facet.field=brand"

# Range фасеты
curl "http://localhost:8983/solr/products/select?q=*:*&facet=true&facet.range=price&facet.range.start=0&facet.range.end=200000&facet.range.gap=50000"

# Query фасеты
curl "http://localhost:8983/solr/products/select?q=*:*&facet=true&facet.query=price:[0 TO 50000]&facet.query=price:[50000 TO 100000]&facet.query=price:[100000 TO *]"

# Pivot фасеты (многоуровневые)
curl "http://localhost:8983/solr/products/select?q=*:*&facet=true&facet.pivot=category,brand"
```

**JSON Facet API** (рекомендуется):

```bash
curl "http://localhost:8983/solr/products/select" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "*:*",
    "facet": {
      "categories": {
        "type": "terms",
        "field": "category",
        "limit": 10,
        "facet": {
          "avg_price": "avg(price)",
          "max_price": "max(price)"
        }
      },
      "price_ranges": {
        "type": "range",
        "field": "price",
        "start": 0,
        "end": 200000,
        "gap": 50000
      }
    }
  }'
```

### Highlighting

```bash
curl "http://localhost:8983/solr/products/select" \
  --data-urlencode "q=iphone" \
  --data-urlencode "hl=true" \
  --data-urlencode "hl.fl=name,description" \
  --data-urlencode "hl.simple.pre=<em>" \
  --data-urlencode "hl.simple.post=</em>" \
  --data-urlencode "hl.fragsize=150"
```

Ответ:

```json
{
  "response": {
    "docs": [
      {"id": "1", "name": "iPhone 15 Pro", "description": "..."}
    ]
  },
  "highlighting": {
    "1": {
      "name": ["<em>iPhone</em> 15 Pro"],
      "description": ["Флагманский смартфон Apple с чипом A17 Pro"]
    }
  }
}
```

### Spell Checking и Suggestions

```xml
<!-- solrconfig.xml -->
<searchComponent name="spellcheck" class="solr.SpellCheckComponent">
    <lst name="spellchecker">
        <str name="name">default</str>
        <str name="field">_text_</str>
        <str name="classname">solr.DirectSolrSpellChecker</str>
    </lst>
</searchComponent>

<requestHandler name="/spell" class="solr.SearchHandler">
    <lst name="defaults">
        <str name="spellcheck">true</str>
        <str name="spellcheck.count">5</str>
    </lst>
    <arr name="last-components">
        <str>spellcheck</str>
    </arr>
</requestHandler>
```

```bash
curl "http://localhost:8983/solr/products/spell?q=iphne&spellcheck=true&spellcheck.collate=true"
```

### Suggester (Автодополнение)

```xml
<!-- solrconfig.xml -->
<searchComponent name="suggest" class="solr.SuggestComponent">
    <lst name="suggester">
        <str name="name">mySuggester</str>
        <str name="lookupImpl">FuzzyLookupFactory</str>
        <str name="dictionaryImpl">DocumentDictionaryFactory</str>
        <str name="field">name</str>
        <str name="suggestAnalyzerFieldType">text_general</str>
        <str name="buildOnStartup">true</str>
    </lst>
</searchComponent>

<requestHandler name="/suggest" class="solr.SearchHandler">
    <lst name="defaults">
        <str name="suggest">true</str>
        <str name="suggest.count">10</str>
        <str name="suggest.dictionary">mySuggester</str>
    </lst>
    <arr name="components">
        <str>suggest</str>
    </arr>
</requestHandler>
```

```bash
curl "http://localhost:8983/solr/products/suggest?suggest.q=iph"
```

---

## Анализаторы и обработка текста

### Цепочка анализа

```
Текст → Char Filters → Tokenizer → Token Filters → Токены
```

### Конфигурация анализатора

```xml
<fieldType name="text_ru" class="solr.TextField" positionIncrementGap="100">
    <analyzer type="index">
        <!-- Char Filters: обработка перед токенизацией -->
        <charFilter class="solr.HTMLStripCharFilterFactory"/>
        <charFilter class="solr.MappingCharFilterFactory" mapping="mapping-FoldToASCII.txt"/>

        <!-- Tokenizer: разбивка на токены -->
        <tokenizer class="solr.StandardTokenizerFactory"/>

        <!-- Token Filters: обработка токенов -->
        <filter class="solr.LowerCaseFilterFactory"/>
        <filter class="solr.StopFilterFactory" words="stopwords_ru.txt" ignoreCase="true"/>
        <filter class="solr.SnowballPorterFilterFactory" language="Russian"/>
        <filter class="solr.RemoveDuplicatesTokenFilterFactory"/>
    </analyzer>

    <analyzer type="query">
        <tokenizer class="solr.StandardTokenizerFactory"/>
        <filter class="solr.LowerCaseFilterFactory"/>
        <filter class="solr.StopFilterFactory" words="stopwords_ru.txt" ignoreCase="true"/>
        <filter class="solr.SynonymGraphFilterFactory" synonyms="synonyms_ru.txt" ignoreCase="true"/>
        <filter class="solr.SnowballPorterFilterFactory" language="Russian"/>
    </analyzer>
</fieldType>
```

### Популярные компоненты анализа

**Tokenizers**:
- `StandardTokenizerFactory` — универсальный, на основе Unicode
- `WhitespaceTokenizerFactory` — разделение по пробелам
- `KeywordTokenizerFactory` — весь текст как один токен
- `PatternTokenizerFactory` — разделение по регулярному выражению
- `ICUTokenizerFactory` — для многоязычного текста

**Token Filters**:
- `LowerCaseFilterFactory` — приведение к нижнему регистру
- `StopFilterFactory` — удаление стоп-слов
- `SnowballPorterFilterFactory` — стемминг (language="Russian")
- `SynonymGraphFilterFactory` — расширение синонимами
- `NGramFilterFactory` — n-граммы
- `EdgeNGramFilterFactory` — префиксные n-граммы
- `ASCIIFoldingFilterFactory` — нормализация диакритических знаков

### Файл синонимов

```text
# synonyms_ru.txt
# Формат: слово1, слово2, слово3 => нормализованное_слово
# или: слово1, слово2 (двустороннее)

телефон, смартфон, мобильный
ноутбук, лаптоп, ноут
компьютер, комп, пк
недорогой, бюджетный, дешевый => доступный
быстрый, скоростной => быстрый
```

### Анализ текста (отладка)

```bash
# Анализ индексации
curl "http://localhost:8983/solr/products/analysis/field?analysis.fieldtype=text_ru&analysis.fieldvalue=Быстрые смартфоны Apple"

# Анализ запроса
curl "http://localhost:8983/solr/products/analysis/field?analysis.fieldtype=text_ru&analysis.fieldvalue=быстрый смартфон&analysis.query=true"
```

---

## Продакшн развёртывание

### Установка и запуск

```bash
# Скачивание
wget https://archive.apache.org/dist/solr/solr/9.4.0/solr-9.4.0.tgz
tar xzf solr-9.4.0.tgz
cd solr-9.4.0

# Запуск standalone
bin/solr start -p 8983

# Запуск с ZooKeeper (SolrCloud)
bin/solr start -c -z localhost:2181

# Создание коллекции
bin/solr create -c products -s 2 -rf 2
```

### Docker Compose

```yaml
version: '3.8'

services:
  zookeeper:
    image: zookeeper:3.8
    ports:
      - "2181:2181"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zookeeper:2888:3888;2181
    volumes:
      - zk_data:/data
      - zk_log:/datalog

  solr1:
    image: solr:9.4
    ports:
      - "8983:8983"
    environment:
      ZK_HOST: zookeeper:2181
      SOLR_HEAP: 2g
    volumes:
      - solr1_data:/var/solr
    depends_on:
      - zookeeper
    command: solr-foreground -c

  solr2:
    image: solr:9.4
    ports:
      - "8984:8983"
    environment:
      ZK_HOST: zookeeper:2181
      SOLR_HEAP: 2g
    volumes:
      - solr2_data:/var/solr
    depends_on:
      - zookeeper
    command: solr-foreground -c

volumes:
  zk_data:
  zk_log:
  solr1_data:
  solr2_data:
```

### JVM настройки

```bash
# solr.in.sh или solr.in.cmd
SOLR_HEAP="4g"
SOLR_JAVA_MEM="-Xms4g -Xmx4g"

# GC настройки для Solr 9+
GC_TUNE="-XX:+UseG1GC \
         -XX:MaxGCPauseMillis=100 \
         -XX:+ParallelRefProcEnabled \
         -XX:+ExplicitGCInvokesConcurrent"

# Off-heap память для DocValues
SOLR_OPTS="$SOLR_OPTS -XX:MaxDirectMemorySize=2g"
```

### Мониторинг

**Встроенные метрики**:

```bash
# Системные метрики
curl "http://localhost:8983/solr/admin/metrics?group=solr.jvm"

# Метрики коллекции
curl "http://localhost:8983/solr/admin/metrics?group=solr.core.products"

# Метрики кэша
curl "http://localhost:8983/solr/admin/metrics?prefix=CACHE"
```

**Prometheus экспортер**:

```xml
<!-- solr.xml -->
<metrics>
    <reporter name="prometheus" class="solr.PrometheusMetricsReporter"/>
</metrics>
```

```bash
curl "http://localhost:8983/solr/admin/metrics?wt=prometheus"
```

### Бэкапы

```bash
# Создание бэкапа коллекции
curl "http://localhost:8983/solr/admin/collections?action=BACKUP&name=products_backup&collection=products&location=/backup/solr"

# Восстановление
curl "http://localhost:8983/solr/admin/collections?action=RESTORE&name=products_backup&collection=products_restored&location=/backup/solr"
```

### Масштабирование

**Добавление шардов**:

```bash
# Split существующего шарда
curl "http://localhost:8983/solr/admin/collections?action=SPLITSHARD&collection=products&shard=shard1"
```

**Добавление реплик**:

```bash
curl "http://localhost:8983/solr/admin/collections?action=ADDREPLICA&collection=products&shard=shard1&node=solr3:8983_solr"
```

---

## Сравнение с Elasticsearch

| Критерий | Solr | Elasticsearch |
|----------|------|---------------|
| **Схема** | Schema-based (строгая) | Schemaless (динамическая) |
| **Запросы** | Lucene syntax, множество парсеров | Query DSL (JSON) |
| **API** | HTTP + JSON/XML | REST + JSON |
| **Админка** | Встроенный UI | Kibana (отдельный продукт) |
| **Координация** | ZooKeeper (внешний) | Встроенный discovery |
| **Streaming** | Streaming Expressions | Aggregations |
| **ML** | Learning to Rank плагин | ML node (платно в X-Pack) |
| **Лицензия** | Apache 2.0 | SSPL / Elastic License |

### Когда выбирать Solr

1. **Строгая схема данных** — когда требуется контроль над типами полей
2. **Сложный текстовый анализ** — богатые возможности настройки анализаторов
3. **Legacy интеграции** — множество плагинов и Data Import Handler
4. **Rich document parsing** — встроенная поддержка Tika для PDF, Office
5. **Полностью open source** — Apache License без ограничений
6. **Streaming analytics** — Parallel SQL и Streaming Expressions

### Когда выбирать Elasticsearch

1. **Быстрый старт** — меньше конфигурации, schemaless
2. **Логирование и APM** — экосистема ELK Stack
3. **Аналитика** — мощные агрегации, Kibana визуализации
4. **Автомасштабирование** — проще управление кластером
5. **Time series** — оптимизирован для временных рядов

---

## Практические примеры

### Поиск товаров с фасетами

```bash
curl "http://localhost:8983/solr/products/select" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "смартфон",
    "filter": ["price:[30000 TO 100000]", "in_stock:true"],
    "fields": ["id", "name", "price", "category", "brand"],
    "sort": "score desc, price asc",
    "limit": 20,
    "facet": {
      "categories": {
        "type": "terms",
        "field": "category"
      },
      "brands": {
        "type": "terms",
        "field": "brand",
        "limit": 10
      },
      "price_ranges": {
        "type": "range",
        "field": "price",
        "start": 0,
        "end": 200000,
        "gap": 25000
      }
    },
    "highlight": {
      "fields": ["name", "description"],
      "pre": "<mark>",
      "post": "</mark>"
    }
  }'
```

### Автодополнение с учётом популярности

```xml
<!-- Поле для suggestions с boost по популярности -->
<field name="suggest_field" type="text_suggest" indexed="true" stored="true"/>
<field name="popularity" type="pint" indexed="true" stored="true"/>

<searchComponent name="suggest" class="solr.SuggestComponent">
    <lst name="suggester">
        <str name="name">productSuggester</str>
        <str name="lookupImpl">BlendedInfixLookupFactory</str>
        <str name="dictionaryImpl">DocumentDictionaryFactory</str>
        <str name="field">suggest_field</str>
        <str name="weightField">popularity</str>
        <str name="blenderType">linear</str>
        <str name="suggestAnalyzerFieldType">text_suggest</str>
    </lst>
</searchComponent>
```

---

## Полезные ресурсы

- [Официальная документация Solr](https://solr.apache.org/guide/)
- [Apache Solr Reference Guide](https://solr.apache.org/guide/solr/latest/)
- [Solr GitHub Repository](https://github.com/apache/solr)
- [Lucene Query Syntax](https://lucene.apache.org/core/9_0_0/queryparser/org/apache/lucene/queryparser/classic/package-summary.html)
- [Solr in Action (книга)](https://www.manning.com/books/solr-in-action)
