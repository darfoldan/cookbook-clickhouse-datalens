## Использование таблиц. Представления и оконные функции

В предыдущих рецептах вы проектировали и оптимизировали базовую таблицу. Теперь время перейти к подготовке производных показателей для использования в Yandex DataLens. В этом рецепте вы создадите представление с оконными функциями и проверите стоимость таких вычислений.

Оконные функции выносятся на сторону БД, чтобы не передавать сырые данные в BI-систему и
не считать их там: в Yandex DataLens встроенные оконные функции реализуются как агрегации
над данными, уже полученными из источника.

Переносить вычисления, в том числе оконные, в источник нужно и в том случае, когда
фильтрация в BI-системе должна выполняться после расчетов. Если результаты оконных
вычислений должны использоваться как измерения, их также стоит переносить в
источник.

### Представление

Представление позволяет реализовать запрос-витрину, который для внешних систем
вроде BI выглядит как обычная таблица, и перенести часть логики на сторону БД —
в том числе вычисления и маскирование полей.

Правило, которым нельзя пренебрегать: в представлениях всем полям задаются явные
алиасы, а таблицам — короткие псевдонимы; в запросах с несколькими таблицами это
обязательно. Без этого имена колонок на выходе легко превращаются в
`alias.field` или `table.field`, и весь датасет в BI-системе ломается.
Полагаться на автоматический вывод имен нельзя: нет никаких гарантий, что завтра
в одной из таблиц не появится поле с таким же названием или что к представлению
не добавят еще одну таблицу.

```sql
-- Представление с оконными функциями
CREATE VIEW financial_transactions_analytics AS
SELECT
    ft.transaction_id    AS transaction_id,
    ft.account_id        AS account_id,
    ft.merchant_category AS merchant_category,
    ft.country           AS country,
    ft.transaction_date  AS transaction_date,
    ft.amount            AS amount,
    ft.status            AS status,
    ft.risk_score        AS risk_score,

    -- Ранг транзакции по сумме внутри категории и страны за месяц
    -- Партиция: ~200 комбинаций (merchant_category × country × month)
    rank() OVER (
        PARTITION BY ft.merchant_category, ft.country,
                     dateTrunc('month', ft.transaction_date)
        ORDER BY ft.amount DESC
    ) AS amount_rank_in_segment,

    -- Доля суммы транзакции от общей суммы по категории и стране
    round(
        ft.amount / sum(ft.amount) OVER (
            PARTITION BY ft.merchant_category, ft.country
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) * 100,
        2
    ) AS pct_of_segment_total,

    -- Скользящая сумма по аккаунту (нарастающий итог по дате)
    -- Партиция: ~1 000 000 аккаунтов — самая дорогая!
    sum(ft.amount) OVER (
        PARTITION BY ft.account_id
        ORDER BY ft.transaction_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS account_running_total,

    -- Предыдущая транзакция аккаунта (lag)
    -- 0, а не 0. — у amount тип Decimal(18, 2), с Float64-литералом нет общего супертипа
    lagInFrame(ft.amount, 1, 0) OVER (
        PARTITION BY ft.account_id
        ORDER BY ft.transaction_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS prev_tx_amount,

    -- Отклонение от средней суммы по категории
    -- Партиция: ~20 категорий
    round(
        ft.amount - avg(ft.amount) OVER (
            PARTITION BY ft.merchant_category
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ),
        2
    ) AS diff_from_category_avg,

    -- Перцентильный ранг по risk_score внутри страны
    -- Партиция: ~20 стран
    percent_rank() OVER (
        PARTITION BY ft.country
        ORDER BY ft.risk_score
    ) AS risk_percentile_in_country

FROM financial_transactions AS ft
WHERE ft.status = 'completed';
```

### Стоимость оконных функций

Ниже — запрос к представлению, который найдет топ транзакций с высоким риском, одновременно входящих в топ-3 по сумме в своем сегменте.
Этот запрос выполняется медленнее предыдущих, потому что в нем запрашиваются результаты сразу нескольких оконных функций.

Причина: оконные функции вычисляются для всех строк, удовлетворяющих
`WHERE status = 'completed'`, и только после этого применяются фильтры
`mount_rank_in_segment <= 3` и `risk_percentile_in_country > 0.9`.

Фильтр по `transaction_date` частично помогает: `PRIMARY KEY` таблицы
позволяет ClickHouse отсечь ненужные гранулы до вычисления окон,
но внутри отобранного диапазона все строки все равно обрабатываются.

Это фундаментальное свойство оконных функций: они требуют полного
прохода по каждой партиции (`PARTITION BY`), прежде чем вернуть результат.

```sql
-- Запрос к представлению
-- Топ транзакций с высоким риском, которые одновременно
-- входят в топ-3 по сумме в своем сегменте

SELECT
    transaction_id,
    account_id,
    merchant_category,
    country,
    transaction_date,
    amount,
    risk_score,
    amount_rank_in_segment,
    pct_of_segment_total,
    risk_percentile_in_country,
    diff_from_category_avg
FROM financial_transactions_analytics
WHERE amount_rank_in_segment <= 3
  AND risk_percentile_in_country > 0.9
  AND transaction_date BETWEEN today() - INTERVAL 1 YEAR AND today()
ORDER BY risk_score DESC, amount DESC
LIMIT 50;
```

> <sub>50 rows in set. Elapsed: 5.473 sec. Processed 50.00 million rows, 1.30 GB (9.14 million rows/s., 237.54 MB/s.)<br>Peak memory usage: 2.09 GiB.</sub>

Вывод подтверждает: запрос на порядки дороже
single-window-запросов из следующего раздела «Запрос с одной оконной функцией», поскольку оконные функции считаются по всем строкам
`status = 'completed'` до применения фильтров `amount_rank_in_segment`
и `risk_percentile_in_country`.

```sql
-- Проверка плана выполнения
EXPLAIN
SELECT
    transaction_id,
    amount_rank_in_segment,
    risk_percentile_in_country
FROM financial_transactions_analytics
WHERE amount_rank_in_segment <= 3
  AND risk_percentile_in_country > 0.9
  AND transaction_date BETWEEN today() - INTERVAL 1 YEAR AND today()
ORDER BY risk_score DESC
LIMIT 50;
```

### Запрос с одной оконной функцией

В BI-системах целесообразно реализовывать не огромные, сложные для анализа таблицы, а
наборы графиков, в которых участвует лишь несколько измерений и показателей.
Если запрос включает лишь одно-два значения из оконной функции — ситуация с
производительностью значительно лучше, чем при обращении к представлению, при котором
вычисляются все оконные функции сразу.

```sql
-- Нарастающий итог по аккаунту
SELECT
    transaction_date,
    amount,
    sum(amount) OVER (
        PARTITION BY account_id
        ORDER BY transaction_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS account_running_total
FROM financial_transactions
WHERE account_id = 282208 -- сделайте выборку для получения реальных сгенерированных значений поля для фильтра
  AND status = 'completed'
ORDER BY transaction_date;
```

> <sub>57 rows in set. Elapsed: 0.064 sec. Processed 50.00 million rows, 252.42 MB (781.25 million rows/s., 3.94 GB/s.)<br>Peak memory usage: 6.13 MiB.</sub>

`EXPLAIN` показывает, что `PRIMARY KEY` не отсекает гранулы по `account_id` —
он стоит четвертым в `ORDER BY` после `merchant_category`, `country` и `month`.
Данные физически не сгруппированы по `account_id`, поэтому ClickHouse®
вынуждена читать большую часть таблицы. Оконная функция при этом все равно вычисляется только для строк
одного аккаунта — `PARTITION BY account_id` гарантирует это логически,
но не снижает объем физически прочитанных данных.

Если запросы по конкретному `account_id` критичны, стоит рассмотреть
вынесение `account_id` на первую позицию в `ORDER BY`, но это противоречит
текущей модели доступа, ориентированной на аналитику по категориям и странам.

```sql

EXPLAIN indexes = 1
SELECT
    transaction_date,
    amount,
    sum(amount) OVER (
        PARTITION BY account_id
        ORDER BY transaction_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS account_running_total
FROM financial_transactions
WHERE account_id = 282208
  AND status = 'completed'
ORDER BY transaction_date;
```

### Селекторы поверх представления

В этом разделе вы проверите использование проекции для запроса под селектор поверх представления —
список уникальных значений одного измерения через `DISTINCT`.
Эта возможность значительно ускоряет работу дашбордов
с обилием селекторов, построенных на представлениях.

Колонку для `DISTINCT` нужно брать не из префикса `ORDER BY` базовой
таблицы. `merchant_category` — первая колонка `ORDER BY`, поэтому чтение
напрямую из `financial_transactions` и так дешево (это `LowCardinality`-
дедупликация без реального скана). Планировщику незачем переключаться
на проекцию — эффект будет незаметен. Колонка `currency` в `ORDER BY` не входит, но
есть в `GROUP BY` проекции `prj_agg_by_category_country` — вот на ней разница
видна.

```sql
-- Проверка использования проекции для запроса под селектор поверх представления —
-- список уникальных значений одного измерения через DISTINCT.

SELECT DISTINCT currency
FROM financial_transactions
ORDER BY currency;
```

> <sub>7 rows in set. Elapsed: 0.013 sec. Processed 47.04 thousand rows, 56.73 KB (3.75 million rows/s., 4.52 MB/s.)<br>Peak memory usage: 620.91 KiB.</sub>

```sql
EXPLAIN indexes = 1
SELECT DISTINCT currency
FROM financial_transactions
ORDER BY currency;

-- Сравните с отключенной проекцией, чтобы увидеть разницу
SELECT DISTINCT currency
FROM financial_transactions
ORDER BY currency
SETTINGS optimize_use_projections = 0;
```

> <sub>7 rows in set. Elapsed: 0.040 sec. Processed 50.00 million rows, 50.01 MB (1.25 billion rows/s., 1.25 GB/s.)<br>Peak memory usage: 4.60 MiB.</sub>

```sql
SELECT DISTINCT currency
FROM financial_transactions
ORDER BY currency;
```

> <sub>7 rows in set. Elapsed: 0.013 sec. Processed 47.38 thousand rows, 57.19 KB (3.64 million rows/s., 4.40 MB/s.)<br>Peak memory usage: 4.61 MiB.</sub>

`EXPLAIN indexes = 1` подтверждает: `ReadFromMergeTree (prj_agg_by_category_country)`,
`Granules: 116/6149` — читается только проекция, а не 50 млн строк базовой
таблицы. В 1000+ раз меньше прочитанных строк и в три раза быстрее по
времени, хотя запрос и так короткий.

```sql
-- Проверьте через query_log, какая проекция была использована
SYSTEM FLUSH LOGS;

SELECT
    query,
    projections,
    formatReadableQuantity(read_rows) AS read_rows,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND arrayExists(t -> t LIKE '%financial_transactions%', tables)
  AND query LIKE '%DISTINCT currency%'
ORDER BY initial_query_start_time DESC
LIMIT 4;
```

> <sub>3 rows in set. Elapsed: 0.033 sec. Processed 275.27 thousand rows, 34.28 MB (8.34 million rows/s., 1.04 GB/s.)<br>Peak memory usage: 33.33 MiB.</sub>

В строке без `SETTINGS optimize_use_projections = 0` поле `projections`
заполнено — `<база>.financial_transactions.prj_agg_by_category_country` —
то же самое, что видно и в `EXPLAIN`.

## Итоги

Вы разобрались, как использовать представления и оконные функции для подготовки производных показателей. 

Что важно запомнить:

- Представления позволяют перенести часть логики на сторону БД, в том числе вычисления.
- Оконные функции требуют полного прохода по каждой партиции `PARTITION BY` перед возвратом результата.
- Запросы с одной-двумя оконными функциями могут работать заметно быстрее, чем запросы к представлению, где вычисляется сразу несколько оконных функций.
