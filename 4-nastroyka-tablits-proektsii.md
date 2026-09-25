# Настройка таблиц: проекции

Базовая таблица из рецепта [Проектирование и подготовка таблиц](ДОБАВИТЬ ССЫЛКУ) оптимизирована под запросы с фильтрацией по `merchant_category` и `country`.
В этом рецепте мы расскажем, как использовать проекции для других сценариев запросов и проверить их влияние на объем чтения.

Продуманной структуры `ORDER BY` и `PRIMARY KEY` хватает для эффективной работы многих BI-систем с небольшим по меркам ClickHouse® объемом данных. Но 
если вариативность фильтрации и объемы данных велики, то использовать только разреженные индексы 
и методы хранения и сжатия будет недостаточно для оптимизации. В таких случаях может помочь использование проекций.

{% cut "Как оценить нагрузку" %}

Чтобы запланировать оптимизацию, рекомендуем ориентироваться не только на объём данных, но и на конкретные сценарии использования. Рассмотрите в качестве примера таблицу из этого рецепта.
Мы предлагаем работать с ней на хосте с 8 CPU и 32 RAM. 
При таких ресурсах после всех манипуляций таблица в 50 миллионов записей будет иметь размер около
2,8 ГБ в несжатом и 1,5 ГБ в сжатом виде. Большинство запросов к ней выполнятся за
доли секунд и лишь при выборке с несколькими оконными функциями время 
выполнения превысит 5 секунд. 

Но в реальности при работе с дашбордом в базу данных уходит сразу 
несколько запросов. Каждый чарт и каждый селектор – это запрос, которому нужны 
ресурсы. К тому же структуры и запросы не всегда эффективны, если соединяются две-три таблицы на 10 миллионов записей каждая, есть самосоединения больших таблиц, семь-восемь уровней подзапросов, оконные функции, лишние сортировки и так далее. Вычисляемые поля в Yandex DataLens добавляют нагрузку на
CPU, уровни агрегации (LOD) реализуются через подзапросы, безопасные конструкции
условий отбора усложняют построение плана анализатором. Все это в сочетании с одновременными запросами дашборда дает большую нагрузку. 


{% endcut %}

Проекция — дополнительный скрытый набор данных внутри таблицы,
поддерживаемый автоматически при вставке. Фактически проекция — это скрытая 
копия данных внутри каждого датапарта, со своим порядком строк, своим первичным 
индексом и, при необходимости, инкрементально пересчитываемыми агрегатами. 
Физически – поддиректория внутри директории датапарта, а не отдельный датапарт. 

Проекции особенно удобны в связке с BI-инструментами: запрос всегда адресован
одной таблице, а ClickHouse® сама выбирает оптимальный тип конструкции `LAYOUT`. В большинстве BI-инструментов смена целевой таблицы нетривиальна. Даже если для этого есть техническая возможность, например через параметры датасета в Yandex DataLens, это влечет ограничения: в таблицах нужно поддерживать полное соответствие состава полей, что создает операционную нагрузку.

Другой способ ускорить выполнение: отдельные таблицы-агрегаты через материализованные
представления (см. [Производные таблицы: материализованные представления](ДОБАВИТЬ ССЫЛКУ)) гарантируют использование нужного типа `LAYOUT` — запрос явно
адресован конкретной таблице, и оптимизатор не принимает решение за вас.
Проекции же используются по усмотрению оптимизатора, и выбор базовой таблицы
вместо проекции — ситуация не такая уж редкая. Обязательно проверяйте, используется ли проекция и насколько эффективно. 

## Что бывает 

- **Альтернативная сортировка** (`normal`). Копия выбранных колонок в другом
  порядке сортировки со своим первичным индексом. Внутренний движок MergeTree
  (нет `GROUP BY`, только `ORDER BY`).
- **Предагрегация** (`aggregate`). Результаты `GROUP BY`, подготовленные заранее.
  Внутренний движок AggregatingMergeTree — назначается автоматически, если в
  проекции есть `GROUP BY`.
- **Индексоподобная** (`_part_offset`). Хранится только ключ сортировки
  и указатель на позицию строки в базовом датапарте; проекция работает как
  вторичный индекс, данные читаются из базовой таблицы. Режимы можно смешивать.
  Начиная с версии 26.1 доступен компактный синтаксис
  `PROJECTION by_town INDEX town TYPE basic`.

## Ограничение: проекция внутри датапарта

Проекция хранится внутри датапартов основной таблицы. Отсюда следствие: ее
сортировка и ее индекс локальны для датапарта, и в лучшем случае читается по одному
диапазону на каждый датапарт, в котором есть подходящие строки. 

- Работает, когда значения ключа проекции сгруппированы: конкретное значение
  встречается не во всех датапартах, а в немногих. Такая группировка возникает из
  корреляции с порядком вставки (типично для дат и монотонных ID) либо с
  основной сортировкой.
- Если ключ проекции распределен по датапартам равномерно и независимо от всего
  остального, высокоселективный фильтр найдет совпадения в каждом датапарте, и
  накладные расходы превысят выигрыш от оптимизации. Это особенно актуально для `_part_offset`-варианта, при котором
  после поиска нужно еще обратиться к базовой таблице.
- Мелкие партиции и много мелких датапартов влияют на проекции сильнее, чем на
  обычные запросы: стоимость умножается на число датапартов. 


## `DISTINCT` и селекторы DataLens

Проекция с предагрегацией используется и для запросов с `DISTINCT`: если
получение всех уникальных значений требует чтения меньшего числа строк из
проекции, ClickHouse® применит ее автоматически. 

Что это означает применительно к Yandex DataLens: каждый селектор на дашборде — это
отдельный запрос с `DISTINCT` по измерению. Селекторов обычно несколько, и такие запросы
выполняются при каждом открытии дашборда, до любых действий пользователя. Для
таблицы в сотни миллионов строк это несколько сканирований на каждое открытие. Операция `DISTINCT` требует много ресурсов, и в глазах пользователя ее выполнение выглядит как загрузка дашборда, хотя сами графики могут быть быстрыми. Задержка особенно заметна на дашбордах с обилием селекторов, построенных на представлениях. 

Проекция с `GROUP BY` по набору измерений, используемых в селекторах, — один из
способов ускорить работу дашборда с минимальными затратами ресурсов. Подробнее об этом способе — в разделе [Селекторы поверх представления](ДОБАВИТЬ ССЫЛКУ).

## Нюансы движков

В движках семейства MergeTree с заменой и агрегацией использование
проекций имеет особенности:

- Настройка `deduplicate_merge_projection_mode` определяет поведение
  проекций при слияниях на ReplacingMergeTree, CollapsingMergeTree,
  VersionedCollapsingMergeTree и при `OPTIMIZE DEDUPLICATE`. Значения:
  `throw` (по умолчанию), `drop`, `rebuild`, `ignore`.
- В режиме по умолчанию добавление проекции на таком движке отклоняется с
  требованием явно выбрать `drop` или `rebuild`. Причина такой осторожности — расхождение результатов: агрегат в проекции может быть
  посчитан по строкам до дедупликации, и тогда значения в проекции и в таблице
  будут отличаться.
- По умолчанию легкие удаления на таблицах с проекциями завершаются ошибкой;
  поведение настраивается через `lightweight_mutation_projection_mode`:
  `throw`, `drop`, `rebuild`.

Практический вывод: на движках с заменой и агрегацией проекции применимы, но
режим нужно выбрать осознанно и проверить совпадение чисел с базовой таблицей
до того, как результат попадет на дашборд. Режим `rebuild` при потоке обновлений может
оказаться дорогим.

## Практика: четыре проекции

Материализация не ретроактивна: добавление проекции действует только на новые
вставки, для существующих данных требуется `MATERIALIZE PROJECTION`. Это тяжелая
операция, прогресс записывается в `system.mutations`.

```sql
-- ------------------------------------------------------------
-- Проекция 1: агрегаты по категории и стране
-- Ускоряет: COUNT, SUM, AVG по измерениям merchant_category/country
-- Внутренний движок: AggregatingMergeTree (автоматически, так как есть GROUP BY)
-- ------------------------------------------------------------
ALTER TABLE financial_transactions
ADD PROJECTION prj_agg_by_category_country
(
    SELECT
        merchant_category,
        country,
        currency,
        status,
        dateTrunc('month', transaction_date) AS month,
        count()                              AS tx_count,
        sum(amount)                          AS total_amount,
        avg(amount)                          AS avg_amount,
        sum(fee)                             AS total_fee,
        sum(cashback_amount)                 AS total_cashback,
        avg(risk_score)                      AS avg_risk_score
    GROUP BY
        merchant_category,
        country,
        currency,
        status,
        month
);

ALTER TABLE financial_transactions
    MATERIALIZE PROJECTION prj_agg_by_category_country;

-- Пример запроса, использующего проекцию
SELECT
    merchant_category,
    country,
    count()            AS total_transactions,
    sum(amount)        AS revenue,
    avg(risk_score)    AS risk
FROM financial_transactions
GROUP BY merchant_category, country
ORDER BY revenue DESC;
```

> <sub>30 rows in set. Elapsed: 0.017 sec. Processed 54.10 thousand rows, 3.61 MB (3.12 million rows/s., 208.32 MB/s.)<br>Peak memory usage: 937.27 KiB.</sub>

```sql
-- Проверка
EXPLAIN indexes = 1
SELECT
    merchant_category,
    country,
    count()        AS total_transactions,
    sum(amount)    AS revenue,
    avg(risk_score) AS risk
FROM financial_transactions
GROUP BY merchant_category, country
ORDER BY revenue DESC;
```

> <sub>В выводе EXPLAIN: ReadFromMergeTree (prj_agg_by_category_country)</sub>

```sql
-- ------------------------------------------------------------
-- Проекция 2: агрегаты по аккаунту
-- Ускоряет: профиль клиента, историю по account_id
-- Внутренний движок: AggregatingMergeTree (автоматически, так как есть GROUP BY)
-- ------------------------------------------------------------
ALTER TABLE financial_transactions
ADD PROJECTION prj_agg_by_account
(
    SELECT
        account_id,
        merchant_category,
        payment_method,
        dateTrunc('month', transaction_date) AS month,
        count()                              AS tx_count,
        sum(amount)                          AS total_amount,
        max(amount)                          AS max_amount,
        sum(cashback_amount)                 AS total_cashback,
        avg(risk_score)                      AS avg_risk_score
    GROUP BY
        account_id,
        merchant_category,
        payment_method,
        month
);

ALTER TABLE financial_transactions
    MATERIALIZE PROJECTION prj_agg_by_account;

-- Пример запроса, использующего проекцию
-- dateTrunc в GROUP BY — ClickHouse сопоставит с алиасом month в проекции
SELECT
    dateTrunc('month', transaction_date) AS month,
    merchant_category,
    count()                              AS transactions,
    sum(amount)                          AS spent
FROM financial_transactions
WHERE account_id = 42345
GROUP BY month, merchant_category
ORDER BY month;
```

> <sub>27 rows in set. Elapsed: 0.031 sec. Processed 937.86 thousand rows, 44.09 MB (30.25 million rows/s., 1.42 GB/s.)<br>Peak memory usage: 11.70 MiB.</sub>

```sql
-- Проверка
EXPLAIN indexes = 1
SELECT
    dateTrunc('month', transaction_date) AS month,
    merchant_category,
    count() AS transactions,
    sum(amount) AS spent
FROM financial_transactions
WHERE account_id = 42345
GROUP BY
    month,
    merchant_category
ORDER BY month;
```

> <sub>В выводе EXPLAIN: ReadFromMergeTree (prj_agg_by_account)</sub>

```sql
-- ------------------------------------------------------------
-- Проекция 3: иная сортировка для фрод-аналитики
-- Ускоряет: ORDER BY risk_score, amount — альтернативный порядок чтения
-- Внутренний движок: MergeTree (нет GROUP BY, только ORDER BY)
-- ВНИМАНИЕ: SELECT * дублирует все данные таблицы на диске!
-- ------------------------------------------------------------
ALTER TABLE financial_transactions
ADD PROJECTION prj_order_by_risk
(
    SELECT *
    ORDER BY (risk_score, amount, transaction_date)
);

ALTER TABLE financial_transactions
    MATERIALIZE PROJECTION prj_order_by_risk;

-- Пример запроса, использующего проекцию
SELECT
    transaction_id,
    account_id,
    country,
    merchant_category,
    amount,
    risk_score
FROM financial_transactions
WHERE risk_score > 90
  AND amount > 500
ORDER BY risk_score DESC, amount DESC
LIMIT 100;
```

> <sub>100 rows in set. Elapsed: 0.053 sec. Processed 5.63 million rows, 94.68 MB (106.19 million rows/s., 1.79 GB/s.)<br>Peak memory usage: 14.27 MiB.</sub>

```sql
-- Проверка
EXPLAIN indexes = 1
SELECT
    transaction_id,
    account_id,
    country,
    merchant_category,
    amount,
    risk_score
FROM financial_transactions
WHERE (risk_score > 90) AND (amount > 500)
ORDER BY
    risk_score DESC,
    amount DESC
LIMIT 100
```

> <sub>В выводе EXPLAIN: ReadFromMergeTree (prj_order_by_risk)</sub>

Проекции-индексы (projection index) хранят только ключ сортировки и виртуальную колонку _part_offset, по которой строки читаются из базовой таблицы. В примере ниже используется компактный синтаксис `INDEX ... TYPE basic`.

```sql
-- ------------------------------------------------------------
-- Проекция 4: lookup по transaction_id (точечный поиск)
-- Ускоряет: WHERE transaction_id = ?
-- Внутренний движок: projection index — хранит только _part_offset
-- Преимущество: минимальное потребление дискового пространства
-- может быть использован при ORDER BY (toDate(created_at), transaction_id)
-- при наличии корреляции между created_at и transaction_id
-- ------------------------------------------------------------
ALTER TABLE financial_transactions
    ADD PROJECTION idx_by_transaction_id INDEX transaction_id TYPE basic;

ALTER TABLE financial_transactions
    MATERIALIZE PROJECTION idx_by_transaction_id;

-- Пример запроса, использующего проекцию
-- подставьте существующее значение transaction_id (диапазон 1..50 000 000)
SELECT
    transaction_id,
    account_id,
    amount,
    status,
    transaction_date
FROM financial_transactions
WHERE transaction_id = 12345678;
```

> <sub>1 rows in set. Elapsed: 0.107 sec. Processed 50.00 million rows, 400.00 MB (467.29 million rows/s., 3.74 GB/s.)<br>Peak memory usage: 5.89 MiB.</sub>

```sql
-- ВАЖНО: Для такого распределения данных projection index на _part_offset не подходит.
-- Механизм эффективен, когда данные естественно кластеризованы по индексируемой колонке внутри датапартов, чего
-- в этом случае нет из-за случайной генерации. Отличный пример того, что работу проекций надо проверять.

-- Можно заставить использовать idx_by_transaction_id явно.
-- Например, расширить саму проекцию нужными колонками, а не оставлять
-- только ключ. Компактный синтаксис `INDEX ... TYPE basic` дает только
-- ключ + _part_offset, но с помощью ADD PROJECTION можно явно перечислить
-- любой набор колонок (необязательно SELECT * целиком);
-- сначала удалить нерабочую проекцию, она занимает место.
ALTER TABLE financial_transactions DROP PROJECTION idx_by_transaction_id;
ALTER TABLE financial_transactions
ADD PROJECTION idx_by_transaction_id
(
    SELECT transaction_id, account_id, amount, status, transaction_date
    ORDER BY transaction_id
);

ALTER TABLE financial_transactions
    MATERIALIZE PROJECTION idx_by_transaction_id;

SELECT transaction_id, account_id, amount, status, transaction_date
FROM financial_transactions
WHERE transaction_id = 12345678;
```

> <sub>1 row in set. Elapsed: 0.012 sec. Processed 303.10 thousand rows, 2.46 MB (25.16 million rows/s., 204.34 MB/s.)<br>Peak memory usage: 1.49 MiB.</sub>

Планировщик выбрал `idx_by_transaction_id`.

## Практика: проверка использования

Проекции занимают место на диске и расходуют ресурсы при слияниях. Не храните те,
что не используются.

```sql
-- ============================================================
-- 5. ПРОВЕРКА РАБОТЫ ПРОЕКЦИЙ
-- ============================================================

-- Метод 1: EXPLAIN — посмотреть ReadFromMergeTree (имя проекции или имя таблицы)
EXPLAIN indexes = 1
SELECT
    transaction_id,
    account_id,
    country,
    merchant_category,
    amount,
    risk_score
FROM financial_transactions
WHERE risk_score > 90
  AND amount > 5000
ORDER BY risk_score DESC, amount DESC
LIMIT 100;
```

```sql
-- Метод 2: FORMAT Null + force_optimize_projection_name
-- Если проекция НЕ используется, запрос завершится с ошибкой
-- Проверить prj_order_by_risk
SELECT
    transaction_id,
    account_id,
    country,
    merchant_category,
    amount,
    risk_score
FROM financial_transactions
WHERE risk_score > 90
  AND amount > 5000
ORDER BY risk_score DESC, amount DESC
LIMIT 100
FORMAT Null
SETTINGS force_optimize_projection_name = 'prj_order_by_risk';
```

Ошибки нет — `force_optimize_projection_name` подтверждает, что `prj_order_by_risk` реально используется:

> <sub>100 rows in set. Elapsed: 0.057 sec. Processed 5.63 million rows, 94.71 MB (98.77 million rows/s., 1.66 GB/s.)<br>Peak memory usage: 15.29 MiB.</sub>

```sql
-- Проверить prj_agg_by_category_country
-- Обратиться к базовым колонкам, как в разделе [Практика: четыре проекции](ДОБАВИТЬ ССЫЛКУ, если можем ставить линк на раздел внутри одного рецепта). ClickHouse сама сопоставит
-- агрегаты с проекцией.
SELECT
    merchant_category,
    country,
    count()         AS total_transactions,
    sum(amount)     AS revenue,
    avg(risk_score) AS risk
FROM financial_transactions
GROUP BY merchant_category, country
ORDER BY revenue DESC
FORMAT Null
SETTINGS force_optimize_projection_name = 'prj_agg_by_category_country';
```

Если проекция не используется, возникнет ошибка вида:

> <sub>Exception: Projection prj_agg_by_category_country is specified in setting force_optimize_projection_name, but it is not used.</sub>

> <sub>30 rows in set. Elapsed: 0.016 sec. Processed 47.38 thousand rows, 3.17 MB (2.96 million rows/s., 197.85 MB/s.)<br>Peak memory usage: 4.84 MiB.</sub>

```sql
-- Метод 3: system.query_log — реальная статистика после выполнения
SELECT
    query,
    projections,
    formatReadableQuantity(read_rows) AS read_rows,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND arrayExists(t -> t LIKE '%financial_transactions%', tables)
ORDER BY initial_query_start_time DESC
LIMIT 10;
```

> <sub>10 rows in set. Elapsed: 0.023 sec. Processed 271.41 thousand rows, 3.78 MB (11.80 million rows/s., 164.38 MB/s.)<br>Peak memory usage: 10.14 MiB.</sub>

Колонка `projections` в выводе явно называет использованную проекцию,
например: `<база>.financial_transactions.prj_agg_by_category_country`. Это прямой способ увидеть выбор оптимизатора постфактум.

## ИТОГИ

Дописать методисту
