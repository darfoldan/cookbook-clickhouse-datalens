## Производные таблицы - материализованные представления

Если рассчитывать производные показатели при каждом обращении к `VIEW` слишком затратно, результаты можно подготовить заранее.

Для реализации классического подхода с таблицами-агрегатами в Clickhouse можно 
использовать материализованные представления и обновляемые материализованные представления.

### Инкрементальный MV — триггер на вставку

Обычный материализованный view в ClickHouse® — *это триггер на вставку*. Он
срабатывает при каждом `INSERT` в исходную таблицу и вычисляет агрегаты в момент
вставки.

Ключевой момент: агрегаты вычисляются над вставляемым блоком, а не над всей
таблицей. Доагрегация выполняется механизмами SummingMergeTree и
AggregatingMergeTree на фоновых мержах.

Отдельное правило, нарушение которого ловится не сразу: колонки MV сопоставляются
с колонками целевой таблицы по именам, а не по позициям.

Осторожно: не рекомендуется использовать в качестве источника
ReplacingMergeTree и CollapsingMergeTree — MV видит строки до фоновой
дедупликации и схлопывания, что может привести к некорректным агрегатам.
Аналогично, при удалении или изменении данных в обычной MergeTree перенос
изменений в целевые таблицы не произойдет.

Внимание: если в запросе представления используется несколько таблиц, то триггер
будет срабатывать при вставке в первую из них и в общем случае не рекомендуется
строить представление на нескольких таблицах, а при необходимости обогащения данных 
следует использовать словари `DICTIONARY`.

```sql
-- ------------------------------------------------------------
-- MV 1: Агрегация по merchant_category + currency за день
-- Ускоряет: аналитику оборота по категориям мерчантов
-- Движок: SummingMergeTree — суммирует числовые колонки
-- при совпадении ключа сортировки
-- ------------------------------------------------------------
-- Целевая таблица
CREATE TABLE financial_transactions_daily_summary
(
    transaction_date  Date,
    merchant_category LowCardinality(String),
    currency          LowCardinality(String),
    total_amount      Decimal(18, 2),
    total_fee         Decimal(10, 4),
    tx_count          UInt64
)
ENGINE = SummingMergeTree
ORDER BY (merchant_category, currency, transaction_date);

-- Материализованное представление
CREATE MATERIALIZED VIEW financial_transactions_daily_summary_mv
TO financial_transactions_daily_summary
AS
SELECT
    transaction_date,
    merchant_category,
    currency,
    sum(amount)  AS total_amount,
    sum(fee)     AS total_fee,
    count()      AS tx_count
FROM financial_transactions
GROUP BY transaction_date, merchant_category, currency;
```

```sql
-- Тестовая вставка — проверяем что MV срабатывает
-- и данные попадают в целевую таблицу
-- ПРИМЕЧАНИЕ: transaction_id взяты 50000001..50000005, а не 1..5 — значения
-- 1..5 уже заняты сквозной генерацией из раздела «Партиционирование» (numbers(50000000)).
INSERT INTO financial_transactions
    (transaction_id, account_id, merchant_category, payment_method,
     country, currency, transaction_date, status,
     amount, fee, exchange_rate, cashback_amount, risk_score)
VALUES
    (50000001, 1001, 'Retail',          'card',     'US', 'USD', today(), 'completed', 150.00, 1.5000, 1.0, 0.75, 10),
    (50000002, 1002, 'Retail',          'transfer', 'DE', 'EUR', today(), 'completed', 200.00, 2.0000, 1.1, 1.00, 20),
    (50000003, 1003, 'Food & Beverage', 'card',     'US', 'USD', today(), 'pending',   75.50,  0.7500, 1.0, 0.38, 5),
    (50000004, 1004, 'Food & Beverage', 'cash',     'FR', 'EUR', today(), 'completed', 50.00,  0.5000, 1.1, 0.25, 15),
    (50000005, 1005, 'Retail',          'crypto',   'US', 'USD', today(), 'failed',    500.00, 5.0000, 1.0, 0.00, 95);

-- Проверяем что данные попали в целевую таблицу
-- FINAL нужен для SummingMergeTree — схлопывает несмерженные части
SELECT
    merchant_category,
    currency,
    sum(total_amount) AS total_amount,
    sum(tx_count)     AS tx_count
FROM financial_transactions_daily_summary
FINAL
WHERE transaction_date = today()
GROUP BY merchant_category, currency
ORDER BY total_amount DESC;
```

> <sub>4 rows in set. Elapsed: 0.004 sec. Processed 4.00 rows, 155.00 B (1.00 thousand rows/s., 38.75 KB/s.)<br>Peak memory usage: 4.23 MiB.</sub>

```sql
-- Backfill исторических данных
-- (если таблица уже содержала данные до создания MV)
INSERT INTO financial_transactions_daily_summary
SELECT
    transaction_date,
    merchant_category,
    currency,
    sum(amount)  AS total_amount,
    sum(fee)     AS total_fee,
    count()      AS tx_count
FROM financial_transactions
GROUP BY transaction_date, merchant_category, currency;
```

### AggregatingMergeTree и состояния агрегатов

Там, где нужны не суммы, а средние, уникальные и прочие агрегаты, целевая
таблица хранит не значения, а частичные состояния: колонки типа
`AggregateFunction(...)`, заполняемые функциями с суффиксом `-State` и читаемые
функциями с суффиксом `-Merge`.

```sql
-- ------------------------------------------------------------
-- MV 2: Фрод-мониторинг — агрегация по risk_score bucket + country
-- Ускоряет: анализ распределения рисков по странам
-- Движок: AggregatingMergeTree — хранит partial aggregate states
-- ------------------------------------------------------------
CREATE TABLE financial_transactions_risk_by_country
(
    country       LowCardinality(String),
    risk_bucket   UInt8,          -- 0=low(0-33), 1=medium(34-66), 2=high(67-100)
    tx_count      AggregateFunction(count),
    avg_amount    AggregateFunction(avg, Decimal(18, 2)),
    avg_risk      AggregateFunction(avg, UInt8)
)
ENGINE = AggregatingMergeTree
ORDER BY (country, risk_bucket);

CREATE MATERIALIZED VIEW financial_transactions_risk_by_country_mv
TO financial_transactions_risk_by_country
AS
SELECT
    country,
    toUInt8(intDiv(risk_score, 34)) AS risk_bucket,
    countState()                    AS tx_count,
    avgState(amount)                AS avg_amount,
    avgState(risk_score)            AS avg_risk
FROM financial_transactions
GROUP BY country, risk_bucket;

-- Backfill существующих данных
INSERT INTO financial_transactions_risk_by_country
SELECT
    country,
    toUInt8(intDiv(risk_score, 34)) AS risk_bucket,
    countState()                    AS tx_count,
    avgState(amount)                AS avg_amount,
    avgState(risk_score)            AS avg_risk
FROM financial_transactions
GROUP BY country, risk_bucket;

-- Пример запроса
SELECT
    country,
    risk_bucket,
    countMerge(tx_count) AS tx_count,
    avgMerge(avg_amount) AS avg_amount,
    avgMerge(avg_risk)   AS avg_risk
FROM financial_transactions_risk_by_country
GROUP BY country, risk_bucket
ORDER BY country, risk_bucket;
```

> <sub>45 rows in set. Elapsed: 0.004 sec. Processed 45.00 rows, 3.49 KB (11.25 thousand rows/s., 872.00 KB/s.)<br>Peak memory usage: 237.45 KiB.</sub>

### Refreshable MV: REPLACE и APPEND

Refreshable MV пересчитывается по расписанию, а не по вставке. Подходит для
отчетов, где допустима небольшая задержка. На `INSERT` он не реагирует вообще.

Поддерживаются два режима записи:

- **REPLACE** (по умолчанию) — каждый refresh атомарно перезаписывает все
  содержимое целевой таблицы, старые данные полностью заменяются новыми.
  Подходит для текущих состояний, топов, lookup-таблиц.
- **APPEND** — каждый refresh добавляет строки в конец таблицы, не удаляя
  старые. Вставка не атомарна, это аналог обычного `INSERT INTO ... SELECT`.
  Подходит для накопления снапшотов, исторических срезов, периодических отчетов.

| | REPLACE (по умолчанию) | APPEND |
|---|---|---|
| Старые данные | удаляются | сохраняются |
| Атомарность | атомарна | не атомарна |
| Подходит для | текущее состояние, топы | снапшоты, история, динамика |

Запрос всегда адресуется целевой таблице, а не самому MV.

```sql
-- ------------------------------------------------------------
-- Refreshable MV 1: Ежедневный отчет по статусам транзакций
-- Режим: REPLACE (по умолчанию) — атомарная полная перезапись
-- Обновляется каждый день в 03:00 UTC
-- Пересчитывает ПОЛНОСТЬЮ — подходит для актуального состояния
-- ВНИМАНИЕ: не реагирует на INSERT, только на расписание!
-- ------------------------------------------------------------
-- Целевая таблица (явная, рекомендуется всегда)
CREATE TABLE financial_transactions_status_report
(
    transaction_date  Date,
    status            Enum8('completed'=1,'pending'=2,'failed'=3,'refunded'=4),
    merchant_category LowCardinality(String),
    country           LowCardinality(String),
    tx_count          UInt64,
    total_amount      Decimal(18, 2),
    avg_amount        Float64,
    total_fee         Decimal(10, 4)
)
ENGINE = MergeTree
ORDER BY (transaction_date, status, merchant_category);

-- Refreshable MV через TO — только определение запроса и расписание
CREATE MATERIALIZED VIEW financial_transactions_status_report_mv
REFRESH EVERY 1 DAY OFFSET 3 HOUR
TO financial_transactions_status_report
AS
SELECT
    transaction_date,
    status,
    merchant_category,
    country,
    count()      AS tx_count,
    sum(amount)  AS total_amount,
    avg(amount)  AS avg_amount,
    sum(fee)     AS total_fee
FROM financial_transactions
WHERE transaction_date >= today() - INTERVAL 90 DAY
GROUP BY transaction_date, status, merchant_category, country;

-- Принудительный refresh для проверки (не ждем расписания)
SYSTEM REFRESH VIEW financial_transactions_status_report_mv;
-- ПРИМЕЧАНИЕ: SYSTEM REFRESH VIEW требует отдельный грант SYSTEM VIEWS,
-- которого у read-only пользователя может не быть. Для примера это не
-- препятствие: Refreshable MV без EMPTY выполняет первый refresh сразу
-- при создании, поэтому данные в целевой таблице уже есть.

-- Запрос идет к ЦЕЛЕВОЙ таблице, не к MV!
SELECT
    status,
    merchant_category,
    sum(tx_count)     AS tx_count,
    sum(total_amount) AS total_amount
FROM financial_transactions_status_report  -- <- целевая таблица
WHERE transaction_date = today() - INTERVAL 5 DAY
GROUP BY status, merchant_category
ORDER BY total_amount DESC;
```

> <sub>4 rows in set. Elapsed: 0.004 sec. Processed 361.00 rows, 7.40 KB (90.25 thousand rows/s., 1.85 MB/s.)<br>Peak memory usage: 4.23 MiB.</sub>

```sql
-- ------------------------------------------------------------
-- Refreshable MV 2: Исторические снапшоты риска по странам
-- Режим: APPEND — каждый refresh ДОБАВЛЯЕТ новый срез, не удаляя старые
-- Обновляется каждый час
-- Позволяет отслеживать динамику показателей во времени
-- ВНИМАНИЕ: вставка НЕ атомарна — аналог обычного INSERT INTO ... SELECT
-- ВНИМАНИЕ: таблица растет неограниченно — нужен TTL или партиционирование!
-- ------------------------------------------------------------
-- Целевая таблица (явная, рекомендуется всегда)
CREATE TABLE financial_transactions_risk_snapshots
(
    snapshot_ts   DateTime,
    country       LowCardinality(String),
    tx_count      UInt64,
    total_amount  Decimal(18, 2),
    avg_risk      Float64,
    high_risk_cnt UInt64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(snapshot_ts)
ORDER BY (snapshot_ts, country)
TTL snapshot_ts + INTERVAL 1 YEAR;

-- Refreshable MV с APPEND через TO
CREATE MATERIALIZED VIEW financial_transactions_risk_snapshots_mv
REFRESH EVERY 1 HOUR
APPEND TO financial_transactions_risk_snapshots  -- <- APPEND + явная целевая таблица
AS
SELECT
    now()                    AS snapshot_ts,
    country,
    count()                  AS tx_count,
    sum(amount)              AS total_amount,
    avg(risk_score)          AS avg_risk,
    countIf(risk_score > 66) AS high_risk_cnt
FROM financial_transactions
WHERE transaction_date >= today() - INTERVAL 7 DAY
GROUP BY country;

-- Принудительный refresh для проверки
SYSTEM REFRESH VIEW financial_transactions_risk_snapshots_mv;
-- ПРИМЕЧАНИЕ: как и в REPLACE-примере выше, первый refresh происходит
-- автоматически при создании MV.

-- Запрос идет к ЦЕЛЕВОЙ таблице, не к MV!
SELECT
    snapshot_ts,
    country,
    avg_risk,
    high_risk_cnt
FROM financial_transactions_risk_snapshots  -- <- целевая таблица
WHERE snapshot_ts >= now() - INTERVAL 24 HOUR
ORDER BY snapshot_ts ASC, country;
```

> <sub>9 rows in set. Elapsed: 0.005 sec. Processed 9.00 rows, 287.00 B (1.80 thousand rows/s., 57.40 KB/s.)<br>Peak memory usage: 4.20 MiB.</sub>

9, а не 15 стран — потому что в срез transaction_date >= today() - INTERVAL 7 DAY
попадают только страны, для которых в сквозной генерации 1.6
(today() - 1095 + rand() % 1095, максимум — «вчера») или в тестовой
вставке из раздела «Инкрементальный MV — триггер на вставку» (today(), страны US/DE/FR) нашлись строки за последнюю неделю.

### Что выбирать

| | Инкрементальный MV | Refreshable MV |
|---|---|---|
| Обновление | при каждом INSERT | по расписанию |
| Задержка данных | реальное время | до следующего refresh |
| JOIN между таблицами | ограниченно | полная поддержка |
| LIMIT и сложные подзапросы | нет | да |
| Петабайтные таблицы | да, инкрементально | полный пересчет |
| Подходит для | агрегации, трансформации | отчеты, топы, сложная аналитика |

