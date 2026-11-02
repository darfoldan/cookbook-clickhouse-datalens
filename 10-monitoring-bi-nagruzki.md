# Мониторинг BI-нагрузки

В этом рецепте вы узнаете, как отслеживать BI-нагрузку. Ключевое правило: оценивайте `total_cpu_sec`, то есть суммарное время всех выполнений одного типа запроса, а не `max_ms` отдельного запроса. Именно суммарная нагрузка показывает, что нагружает систему, когда BI-система присылает пачки запросов.

```sql
-- Текущие активные запросы прямо сейчас
-- Что выполняется в данный момент, сгруппировано по типу нагрузки
SELECT
    user,
    count()                                AS query_count,
    sum(elapsed)                           AS total_elapsed_sec,
    max(elapsed)                           AS max_elapsed_sec,
    formatReadableSize(sum(memory_usage))  AS total_memory,
    formatReadableSize(max(memory_usage))  AS max_memory,
    formatReadableQuantity(sum(read_rows)) AS total_read_rows,
    groupArray(left(query, 80))            AS sample_queries
FROM system.processes
GROUP BY user
ORDER BY total_elapsed_sec DESC;
```

> <sub>1 row in set. Elapsed: 0.004 sec. Processed 1.00 rows, 3.38 KB (250.00 rows/s., 843.75 KB/s.)<br>Peak memory usage: 4.02 MiB.</sub>

Если других активных сессий нет, в выводе будет только сам этот запрос: `system.processes` показывает срез на текущий момент, а не историю.

```sql
-- Каких ресурсов не хватает системе: CPU, ввод-вывод, память
-- Агрегированная статистика за последние N минут
-- Показывает узкие места по типам запросов
SELECT
    toStartOfMinute(query_start_time)              AS minute,
    count()                                        AS total_queries,
    countIf(query_duration_ms > 1000)              AS slow_queries,
    countIf(hasAny(tables, [currentDatabase() || '.financial_transactions'])) AS ft_queries,

    -- CPU
    formatReadableQuantity(sum(read_rows))         AS rows_read,
    round(avg(query_duration_ms))                  AS avg_duration_ms,
    max(query_duration_ms)                         AS max_duration_ms,

    -- IO
    formatReadableSize(sum(read_bytes))            AS bytes_read,
    formatReadableSize(sum(written_bytes))         AS bytes_written,

    -- Memory
    formatReadableSize(max(memory_usage))          AS peak_memory,
    formatReadableSize(sum(memory_usage))          AS total_memory,

    -- Параллелизм: сколько запросов шло одновременно
    max(ProfileEvents['ConcurrentQueriesCount'])   AS max_concurrent
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_start_time >= now() - INTERVAL 30 MINUTE
  AND query_start_time < now()
GROUP BY minute
ORDER BY minute DESC;
```

> <sub>31 row in set. Elapsed: 0.016 sec. Processed 272.38 thousand rows, 1.48 MB (17.02 million rows/s., 92.20 MB/s.)<br>Peak memory usage: 4.49 MiB.</sub>

`hasAny()` ищет точное совпадение строки, а в `system.query_log` таблица записана с префиксом БД. Поэтому сравнивать нужно с полным именем `(currentDatabase() || '.financial_transactions')`, иначе `ft_queries` всегда будет `0`.

```sql
-- Топ тяжелых запросов за период
-- Найти запросы, которые суммарно потребляют больше всего ресурсов
-- normalized_query_hash группирует похожие запросы с разными параметрами
SELECT
    normalized_query_hash,
    count()                                AS executions,
    any(left(query, 120))                  AS sample_query,
    round(avg(query_duration_ms))          AS avg_ms,
    max(query_duration_ms)                 AS max_ms,
    round(sum(query_duration_ms) / 1000)   AS total_cpu_sec,
    formatReadableSize(avg(memory_usage))  AS avg_memory,
    formatReadableSize(max(memory_usage))  AS max_memory,
    formatReadableQuantity(avg(read_rows)) AS avg_rows_read,
    formatReadableSize(avg(read_bytes))    AS avg_bytes_read,
    countIf(exception != '')               AS error_count
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_start_time >= now() - INTERVAL 1 HOUR
  AND user != 'system'
GROUP BY normalized_query_hash
ORDER BY total_cpu_sec DESC
LIMIT 20;
```

> <sub>20 rows in set. Elapsed: 0.020 sec. Processed 271.90 thousand rows, 3.20 MB (13.60 million rows/s., 160.11 MB/s.)<br>Peak memory usage: 5.31 MiB.</sub>

На распределенном инстансе Yandex Managed Service for ClickHouse® в топ-20 по `total_cpu_sec` попадают и запросы других пользователей кластера. Эта метрика нужна именно для того, чтобы такие запросы не терялись за `max_ms` одного запроса. Строки, относящиеся к примерам этого рецепта, выглядят так:

- `avg_ms=5473`, `max_memory=2.09 GiB`, `avg_rows_read=50.00 million` — запрос из раздела [Стоимость оконных функций](07-ispolzovanie-tablits-predstavleniya-i-okonnye-funktsii.md#стоимость-оконных-функций) (представление и несколько оконных функций)
- `avg_ms=1153`, `avg_rows_read=50.00 million` — `dictGet` без индексов пропуска данных из раздела [Практика: замеры производительности](06-ispolzovanie-tablits-join-i-slovari.md#практика-замеры-производительности)
- `avg_ms=804`, `avg_rows_read=50.00 million` — backfill `INSERT` в разделе [Инкрементальное MV — триггер на вставку](08-proizvodnye-tablitsy-materializovannye-predstavleniya.md#инкрементальное-mv--триггер-на-вставку)

```sql
-- Конкурентность: пиковая нагрузка по временным слотам
-- Видны моменты, когда BI-система присылает пачки запросов одновременно
SELECT
    toStartOfMinute(query_start_time)                   AS minute,
    count()                                             AS queries_started,
    countIf(hasAny(tables, [currentDatabase() || '.financial_transactions'])) AS on_main_table,
    -- Запросы с подзапросами обычно длиннее и тяжелее
    countIf(query_duration_ms > 2000)                   AS heavy_queries,
    countIf(query_duration_ms <= 500)                   AS fast_queries,
    formatReadableSize(sum(read_bytes))                 AS total_io
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_start_time >= now() - INTERVAL 2 HOUR
  AND user != 'system'
GROUP BY minute
ORDER BY minute DESC;
```

> <sub>121 row in set. Elapsed: 0.013 sec. Processed 272.42 thousand rows, 1.84 MB (20.96 million rows/s., 141.25 MB/s.)<br>Peak memory usage: 4.41 MiB.</sub>

На распределенном инстансе Managed Service for ClickHouse® в выводе видна устойчивая фоновая активность (десятки запросов в минуту), не связанная с примерами рецепта. Учитывайте это при интерпретации пиковой нагрузки: без фильтра по своему пользователю или по своей таблице вывод смешивает вашу нагрузку с чужой.

```sql
-- Ожидания на слияниях и вставках (конкуренция за ресурсы)
-- Если BI-запросы конкурируют с фоновыми слияниями
SELECT
    event_time,
    metric,
    value
FROM system.metric_log
ARRAY JOIN
    ['BackgroundMergesAndMutationsPoolTask',
     'BackgroundMergesAndMutationsPoolSize',
     'MemoryTracking',
     'Query',
     'MergeTreeDataSelectExecutorThreads'] AS metric,
    [CurrentMetric_BackgroundMergesAndMutationsPoolTask,
     CurrentMetric_BackgroundMergesAndMutationsPoolSize,
     CurrentMetric_MemoryTracking,
     CurrentMetric_Query,
     CurrentMetric_MergeTreeDataSelectExecutorThreads] AS value
WHERE event_time >= now() - INTERVAL 10 MINUTE
ORDER BY event_time DESC, metric ASC
LIMIT 10;
```

## Очистка тестовых структур

```sql
-- Шаг 1: удалите все материализованные представления (зависимые объекты первыми)
DROP VIEW IF EXISTS financial_transactions_daily_summary_mv;
DROP VIEW IF EXISTS financial_transactions_risk_by_country_mv;
DROP VIEW IF EXISTS financial_transactions_risk_snapshots_mv;
DROP VIEW IF EXISTS financial_transactions_status_report_mv;

-- Шаг 2: удалите целевые таблицы
DROP TABLE IF EXISTS financial_transactions_daily_summary;
DROP TABLE IF EXISTS financial_transactions_risk_by_country;
DROP TABLE IF EXISTS financial_transactions_risk_snapshots;
DROP TABLE IF EXISTS financial_transactions_status_report;
DROP TABLE IF EXISTS financial_transactions_analytics;
DROP TABLE IF EXISTS financial_transactions_top_by_country;

-- Шаг 3: удалите словари
DROP DICTIONARY IF EXISTS merchant_category_dict;
DROP TABLE IF EXISTS merchant_category_dict_source;

-- Шаг 4: последней удалите исходную таблицу
DROP TABLE IF EXISTS financial_transactions;
```

## Итоги

Вы разобрались, как отслеживать BI-нагрузку в ClickHouse®.

Что важно запомнить:

- Оценивайте `total_cpu_sec` (суммарное время всех выполнений одного типа запроса), не только `max_ms` отдельного запроса.
- `system.processes` показывает активные запросы в текущий момент, `system.query_log` позволяет анализировать историю нагрузки.
- Тяжелые запросы нужно искать по суммарному потреблению ресурсов, группируя похожие запросы по `normalized_query_hash`.
- При анализе нагрузки фильтруйте данные по своему пользователю или таблице, чтобы не смешивать свою нагрузку с чужой.

---

Будем благодарны за обратную связь — это поможет нам улучшить рецепты. Опрос займет не более трех минут. [Пройти опрос](https://forms.yandex.ru/surveys/13904409.de2ec464d13f519954a1fab7fa333720a58d7c10/)
