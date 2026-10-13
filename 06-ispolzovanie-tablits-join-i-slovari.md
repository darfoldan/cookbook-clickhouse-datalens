# Использование таблиц. JOIN и словари

Не все данные, которые необходимы для аналитики, обязательно хранятся в основной широкой таблице. В этом разделе вы узнаете, как обогащать ее справочными данными с помощью `JOIN` и словарей.

ClickHouse® постоянно повышает производительность операций `JOIN`, и эту СУБД уже нельзя считать неподходящей для таких операций. Чаще
всего критичное падение производительности происходит из-за грубых ошибок в структуре соединений и из-за того, что объемы объединяемых данных непропорциональны ресурсам.

## Справочник и внешний ключ

Типовой сценарий — дообогащение данных семантикой — используется и на широких таблицах.

```sql
-- Небольшая таблица: category_id → семантика
CREATE TABLE merchant_category_dict_source
(
    category_id   UInt32,
    category_name LowCardinality(String), -- совпадает с merchant_category в основной таблице
    risk_tier     Enum8(
                      'low'    = 1,
                      'medium' = 2,
                      'high'   = 3
                  ),
    is_regulated  UInt8                   -- 1 = регулируемая отрасль
)
ENGINE = MergeTree
ORDER BY category_id;

INSERT INTO merchant_category_dict_source VALUES
    (1,  'Retail',             'low',    0),
    (2,  'Food & Beverage',    'low',    0),
    (3,  'Travel',             'medium', 0),
    (4,  'Healthcare',         'high',   1),
    (5,  'Entertainment',      'low',    0),
    (6,  'Utilities',          'medium', 1),
    (7,  'E-commerce',         'medium', 0),
    (8,  'Finance',            'high',   1),
    (9,  'Education',          'low',    1),
    (10, 'Automotive',         'medium', 0),
    (11, 'Insurance',          'high',   1),
    (12, 'Real Estate',        'high',   1),
    (13, 'Telecommunications', 'medium', 1),
    (14, 'Gaming',             'medium', 0),
    (15, 'Gambling',           'high',   1),
    (16, 'Pharmaceuticals',    'high',   1),
    (17, 'Logistics',          'medium', 0),
    (18, 'Agriculture',        'low',    0),
    (19, 'Media',              'low',    0),
    (20, 'Government',         'medium', 1);
```

```sql
-- Добавьте внешний ключ в основную таблицу
-- Колонка с маппингом merchant_category → category_id
ALTER TABLE financial_transactions
    ADD COLUMN category_id UInt32 DEFAULT
        toUInt32(indexOf(
            ['Retail', 'Food & Beverage', 'Travel', 'Healthcare',
             'Entertainment', 'Utilities', 'E-commerce', 'Finance',
             'Education', 'Automotive', 'Insurance', 'Real Estate',
             'Telecommunications', 'Gaming', 'Gambling',
             'Pharmaceuticals', 'Logistics', 'Agriculture', 'Media', 'Government'],
            merchant_category
        ));

-- Материализуйте для существующих строк
ALTER TABLE financial_transactions
    MATERIALIZE COLUMN category_id;

-- Проверьте, что маппинг корректный
SELECT
    merchant_category,
    category_id,
    count() AS cnt
FROM financial_transactions
GROUP BY merchant_category, category_id
ORDER BY category_id;
```

> <sub>10 rows in set. Elapsed: 1.367 sec. Processed 50.00 million rows, 50.01 MB (36.58 million rows/s., 36.59 MB/s.)<br>Peak memory usage: 34.72 MiB.</sub>

Эффективность `bloom_filter` напрямую зависит от корреляции
индексируемой колонки с `ORDER BY` таблицы.
`merchant_category` — первый ключ `ORDER BY`, а `category_id` —
ее прямое производное (маппинг 1:1). Данные физически сгруппированы
на диске по `category_id` — все строки с `category_id = 8` (Finance)
лежат рядом в небольшом числе гранул. Механизм `bloom_filter` в такой ситуации максимально эффективен: он быстро
отсекает гранулы, в которых нужного `category_id` заведомо нет.

Если бы `category_id` был случайным (без корреляции с `ORDER BY`), 
значения были бы распределены по всем гранулам и `bloom_filter`
почти не давал бы выигрыша.

```sql
-- bloom_filter на category_id

ALTER TABLE financial_transactions
    ADD INDEX category_id_bf category_id
    TYPE bloom_filter(0.01) GRANULARITY 1;

ALTER TABLE financial_transactions
    MATERIALIZE INDEX category_id_bf;

-- Проверьте, что индекс материализован
SELECT
    name,
    formatReadableSize(data_compressed_bytes)   AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    marks_bytes
FROM system.data_skipping_indices
WHERE table = 'financial_transactions'
  AND name = 'category_id_bf';
```

> <sub>1 rows in set. Elapsed: 0.004 sec. Processed 16.00 rows, 1.20 KB (4.00 thousand rows/s., 299.75 KB/s.)<br>Peak memory usage: 4.02 MiB.</sub>

## Словарь как метод присоединения семантики

Создайте словарь на основе таблицы `merchant_category_dict_source`:

```sql
CREATE DICTIONARY merchant_category_dict
(
    category_id   UInt32,
    category_name String,
    risk_tier     String,
    is_regulated  UInt8
)
PRIMARY KEY category_id
SOURCE(CLICKHOUSE(
    TABLE 'merchant_category_dict_source'
    DB currentDatabase()
))
LAYOUT(FLAT())          -- FLAT — самый быстрый LAYOUT, ключи UInt64, таблица маленькая
LIFETIME(MIN 300 MAX 900);
```

## `JOIN` и `dictGet`

Рассмотрите два варианта использования и сравните количество обработанных строк.

**Вариант A:** обычный `JOIN`

ClickHouse® строит хеш-таблицу из `merchant_category_dict_source`,
затем проверяет каждую строку `financial_transactions`.
Фильтр по `risk_tier` применяется после `JOIN`.
ClickHouse® автоматически строит runtime `bloom_filter`
из правой части `JOIN` (`merchant_category_dict_source`) и передает его
в сканирование левой части (`financial_transactions`) еще до `JOIN`.
Это позволяет отсечь строки `financial_transactions`, у которых `category_id`
заведомо не встречается в правой части, то есть частично имитирует
поведение, которое раньше было доступно только через `dictGet`.
Тем не менее `dictGet` остается быстрее: словарь уже в памяти,
хеш-таблица не строится заново при каждом запросе и статический
`bloom_filter` на `category_id` работает на уровне гранул до чтения данных.

```sql
-- Вариант A: обычный JOIN
SELECT
    ft.merchant_category AS merchant_category,
    ft.country           AS country,
    count()              AS tx_count,
    sum(ft.amount)       AS total_amount,
    avg(ft.risk_score)   AS avg_risk
FROM financial_transactions AS ft
INNER JOIN merchant_category_dict_source AS mcd ON ft.category_id = mcd.category_id
WHERE mcd.risk_tier = 'high'
  AND ft.transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY ft.merchant_category, ft.country
ORDER BY total_amount DESC;
```

> <sub>6 rows in set. Elapsed: 1.256 sec. Processed 50.00 million rows, 650.02 MB (39.81 million rows/s., 517.53 MB/s.)<br>Peak memory usage: 55.57 MiB.</sub>

**Вариант B:** `dictGet` — `Direct Join`

ClickHouse® делает lookup прямо в памяти для каждой строки,
пережившей фильтр по `PRIMARY KEY`.
Фильтр по `risk_tier` вычисляется inline через `dictGet`,
`bloom_filter` на `category_id` отсекает ненужные гранулы.

```sql
-- Вариант B: dictGet — Direct Join
SELECT
    merchant_category,
    country,
    count()            AS tx_count,
    sum(amount)        AS total_amount,
    avg(risk_score)    AS avg_risk
FROM financial_transactions
WHERE dictGet('merchant_category_dict', 'risk_tier', category_id) = 'high'
  AND transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY merchant_category, country
ORDER BY total_amount DESC;
```

> <sub>6 rows in set. Elapsed: 0.247 sec. Processed 11.69 million rows, 198.68 MB (47.31 million rows/s., 804.36 MB/s.)<br>Peak memory usage: 51.05 MiB.</sub>

Сравнение с вариантом A (обычный `JOIN`): 11,69 млн строк против 50,00 млн — `bloom_filter`
на `category_id` отсекает лишние гранулы еще до lookup. Запрос быстрее
примерно в 5 раз: 0,247 с против 1,256 с.

Почему `dictGet` быстрее `JOIN` в этом сценарии:

| | JOIN | dictGet |
|---|---|---|
| Построение хеш-таблицы | Да, каждый раз | Не нужно — словарь уже в RAM |
| Алгоритм | `Hash Join` | `Direct Join` |
| Фильтр по `risk_tier` | После `JOIN` | Inline, строка за строкой |
| `bloom_filter` на `category_id` | Не используется | Отсекает гранулы до lookup |
| Память | Хеш-таблица правой части | Только словарь (уже загружен) |

Подробнее о том, как ускорить работу с помощью словарей, читайте в [документации ClickHouse®](https://clickhouse.com/docs/concepts/features/dictionaries#speeding-up-joins-using-a-dictionary).

## Практика: замеры производительности

При сравнении производительности для более точных результатов можно сбрасывать
кеши перед очередной итерацией:

```sql
-- Сбросить mark cache (индексные метки MergeTree)
SYSTEM DROP MARK CACHE;

-- Сбросить uncompressed cache (несжатые данные)
SYSTEM DROP UNCOMPRESSED CACHE;

-- Сбросить query cache (результаты запросов)
SYSTEM CLEAR QUERY CACHE;

-- Сбросить filesystem cache (кеш над S3 / Azure / локальными дисками)
SYSTEM DROP FILESYSTEM CACHE;

-- Сбросить userspace page cache
SYSTEM DROP PAGE CACHE;
```

```sql
-- План для JOIN
EXPLAIN indexes = 1
SELECT
    ft.merchant_category AS merchant_category,
    ft.country           AS country,
    count()              AS tx_count,
    sum(ft.amount)       AS total_amount
FROM financial_transactions AS ft
INNER JOIN merchant_category_dict_source AS mcd ON ft.category_id = mcd.category_id
WHERE mcd.risk_tier = 'high'
  AND ft.transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY ft.merchant_category, ft.country
ORDER BY total_amount DESC;

-- План для dictGet
EXPLAIN indexes = 1
SELECT
    merchant_category,
    country,
    count()          AS tx_count,
    sum(amount)      AS total_amount
FROM financial_transactions
WHERE dictGet('merchant_category_dict', 'risk_tier', category_id) = 'high'
  AND transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY merchant_category, country
ORDER BY total_amount DESC;
```

```sql
-- Замер времени
-- Baseline: JOIN без индексов
SELECT
    ft.merchant_category AS merchant_category,
    ft.country           AS country,
    count()              AS tx_count,
    sum(ft.amount)       AS total_amount
FROM financial_transactions AS ft
INNER JOIN merchant_category_dict_source AS mcd ON ft.category_id = mcd.category_id
WHERE mcd.risk_tier = 'high'
  AND ft.transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY ft.merchant_category, ft.country
FORMAT Null
SETTINGS use_query_condition_cache = 0;
```

> <sub>6 rows in set. Elapsed: 1.246 sec. Processed 50.00 million rows, 600.02 MB (40.13 million rows/s., 481.55 MB/s.)<br>Peak memory usage: 41.28 MiB.</sub>

```sql
-- Кеши не сбрасывались перед замером — SYSTEM DROP CACHE недоступен
-- read-only-пользователю и является операцией инстанс-wide 

-- dictGet с bloom_filter
SELECT
    merchant_category AS merchant_category,
    country           AS country,
    count()           AS tx_count,
    sum(amount)       AS total_amount
FROM financial_transactions
WHERE dictGet('merchant_category_dict', 'risk_tier', category_id) = 'high'
  AND transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY merchant_category, country
FORMAT Null
SETTINGS use_query_condition_cache = 0;
```

> <sub>6 rows in set. Elapsed: 0.219 sec. Processed 11.69 million rows, 186.99 MB (53.36 million rows/s., 853.85 MB/s.)<br>Peak memory usage: 44.40 MiB.</sub>

```sql
-- dictGet без bloom_filter — для чистоты сравнения
SELECT
    merchant_category AS merchant_category,
    country           AS country,
    count()           AS tx_count,
    sum(amount)       AS total_amount
FROM financial_transactions
WHERE dictGet('merchant_category_dict', 'risk_tier', category_id) = 'high'
  AND transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY merchant_category, country
FORMAT Null
SETTINGS use_query_condition_cache = 0, use_skip_indexes = 0;
```

> <sub>6 rows in set. Elapsed: 1.153 sec. Processed 50.00 million rows, 600.02 MB (43.37 million rows/s., 520.40 MB/s.)<br>Peak memory usage: 4.86 MiB.</sub>

Без индексов пропуска данных `dictGet` читает все 50 млн строк, как и `JOIN`, а не 11,69 млн. Разница во времени против предыдущего замера (0,219 с) подтверждает вклад
`bloom_filter` на `category_id`, а не самого `dictGet`.

```sql
-- Проверка через query_log
SELECT
    query,
    projections,
    formatReadableQuantity(read_rows)  AS read_rows,
    formatReadableQuantity(read_bytes) AS read_bytes,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND arrayExists(t -> t LIKE '%financial_transactions%', tables)
  AND query_duration_ms > 0
ORDER BY initial_query_start_time DESC
LIMIT 6;
```

> <sub>6 rows in set. Elapsed: 0.015 sec. Processed 272.69 thousand rows, 4.25 MB (18.18 million rows/s., 283.08 MB/s.)<br>Peak memory usage: 4.94 MiB.</sub>

В колонке `projections` видно: `JOIN`-вариант читался через `prj_order_by_risk`,
а оба `dictGet`-варианта — напрямую из базовой таблицы (`[]` в `projections`).
Наличие подходящей проекции не гарантирует ее использование именно там,
где ожидается. Это еще один повод для обязательной проверки из раздела «Практика: проверка использования» рецепта [Настройка таблиц. Проекции](ДОБАВИТЬ ССЫЛКУ).

## Итоги

Вы разобрались, как обогащать широкую таблицу справочными данными с помощью `JOIN` и словарей. 

Что важно запомнить:

- Падение производительности `JOIN` чаще всего связано с ошибками в структуре соединений и объемами данных, которые непропорциональны ресурсам.
- При использовании `dictGet` словарь уже находится в памяти, поэтому хеш-таблица не строится заново при каждом запросе.
- Производительность `JOIN` и `dictGet` проверяйте на конкретных запросах с помощью `EXPLAIN` и замеров производительности.


