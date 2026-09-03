## Настройка таблиц - скип-индексы

### Когда работают

В этом разделе рассмотрим skip-индексы как дополнительный способ оптимизации фильтрации и на практике проверим их эффективность для разных сценариев.

Скип-индексы — дополнение к проекциям и сортировке, а не замена. Основная
область применения: колонки невысокой кардинальности, с хорошей корреляцией с
`ORDER BY` таблицы и неравномерным распределением значений, значительные пропуски. 
Тогда индекс быстро отсекает гранулы, где нужного значения заведомо нет.

Если бы индексируемая колонка была случайной (без корреляции с `ORDER BY`),
значения оказались бы размазаны по всем гранулам, и индекс пропуска почти не давал
бы выигрыша.

### Практика: minmax индексы для числовых колонок
Хорошие кандидаты:
`transaction_id` — монотонно возрастающий UInt64. Minmax будет очень эффективен, так как значения естественно упорядочены внутри частей.

Сомнительные и плохие кандидаты:
`account_id` — высокая кардинальность (~1M), но значения внутри гранул будут перемешаны (последний элемент `ORDER BY`). Minmax даст слабую селективность — диапазоны min/max в каждой грануле будут широкими и перекрывающимися.

`fee`, `cashback_amount`, `exchange_rate` — зависят от `amount` и `merchant_category`, корреляция с `ORDER BY` слабая. `Minmax` может помочь, но эффект непредсказуем без реального тестирования.
`payment_method`, `status` — всего 4 значения. `Minmax` будет хранить min=1, max=4 почти в каждой грануле — индекс никогда ничего не отсечет.
https://clickhouse.com/docs/clickstack/managing/performance-tuning#when-to-add-skip-indexes

```sql
-- transaction_id — монотонно возрастает, minmax очень эффективен
-- Особенно полезен для запросов вида WHERE transaction_id BETWEEN X AND Y

-- Сначала удалим конкурирующую проекцию
ALTER TABLE financial_transactions
    DROP PROJECTION idx_by_transaction_id;
-- и создадим индекс
ALTER TABLE financial_transactions
    ADD INDEX idx_minmax_transaction_id transaction_id TYPE minmax GRANULARITY 1;

-- Материализация индексов (обязательно после ADD INDEX!)
ALTER TABLE financial_transactions MATERIALIZE INDEX idx_minmax_transaction_id;

-- Проверяем что индекс реально отсекает гранулы
EXPLAIN indexes = 1
SELECT count(), sum(amount)
FROM financial_transactions
WHERE transaction_id = 97234;
```

В выводе ищем секцию `Skip` — чем меньше левое число в `Parts`, тем лучше работает индекс:

> <sub>Parts: 37/114<br>Granules: 2949/6149</sub>

Если индекс не отсекает парты, то, вероятно, значения transaction_id внутри партов не монотонны — то есть порядок вставки не коррелирует с ключом сортировки таблицы.

### Практика: bloom filter и OR по двум колонкам

Частный, но интересный случай — фильтр `OR` по двум колонкам: найти транзакции,
где аккаунт участвует в любой роли (отправитель ИЛИ получатель), аналог
`credit_account OR debit_account`. Ключом такой запрос не закрывается.

```sql
-- ============================================================
-- 6. BLOOM FILTER ИНДЕКСЫ ДЛЯ OR-ФИЛЬТРАЦИИ ПО ДВУМ КОЛОНКАМ
-- ============================================================
-- Паттерн: найти транзакции где аккаунт участвует в любой роли
-- (отправитель ИЛИ получатель) — аналог credit_account OR debit_account

-- Добавляем колонки
ALTER TABLE financial_transactions
    ADD COLUMN sender_account_id   UInt32 DEFAULT account_id,
    ADD COLUMN receiver_account_id UInt32 DEFAULT toUInt32(rand() % 1000000);

-- Материализуем значения в существующих строках
-- (DEFAULT заполняет только новые вставки, не существующие строки)
ALTER TABLE financial_transactions
    MATERIALIZE COLUMN sender_account_id;

ALTER TABLE financial_transactions
    MATERIALIZE COLUMN receiver_account_id;

-- Добавляем bloom filter индексы
-- false_positive_rate = 0.01: 1% ложных срабатываний
-- GRANULARITY 1: индекс на каждую гранулу (8192 строки)
ALTER TABLE financial_transactions
    ADD INDEX sender_account_bf sender_account_id
    TYPE bloom_filter(0.01) GRANULARITY 1;

ALTER TABLE financial_transactions
    ADD INDEX receiver_account_bf receiver_account_id
    TYPE bloom_filter(0.01) GRANULARITY 1;

-- Материализуем для существующих данных
ALTER TABLE financial_transactions
    MATERIALIZE INDEX sender_account_bf;

ALTER TABLE financial_transactions
    MATERIALIZE INDEX receiver_account_bf;

-- Проверяем что индексы материализованы
SELECT
    name,
    formatReadableSize(data_compressed_bytes)   AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    marks_bytes
FROM system.data_skipping_indices
WHERE table = 'financial_transactions'
  AND name IN ('sender_account_bf', 'receiver_account_bf');
```

> <sub>2 rows in set. Elapsed: 0.078 sec. Processed 15.00 rows, 1.12 KB (192.31 rows/s., 14.40 KB/s.)<br>Peak memory usage: 4.02 MiB.</sub>

```sql
-- Основной запрос: OR по двум колонкам
-- Найти все транзакции где аккаунт 42345 участвует в любой роли
-- Автоматическое использование обоих bloom filter для OR появилось в версии
-- 25.12.
--
-- На более старых версиях OR между разными колонками не позволяет
-- использовать skip indexes — в этом случае переписываем через UNION ALL:
--
--   SELECT ... FROM financial_transactions
--   WHERE sender_account_id = 42345
--     AND transaction_date BETWEEN today() - INTERVAL 1 YEAR AND today()
--   UNION ALL
--   SELECT ... FROM financial_transactions
--   WHERE receiver_account_id = 42345
--     AND transaction_date BETWEEN today() - INTERVAL 1 YEAR AND today()
--
-- Каждая ветка UNION ALL выполняется независимо и использует свой
-- bloom filter — sender_account_bf и receiver_account_bf соответственно.
--
-- UNION ALL и параллелизм: ветки выполняются одновременно, но общий пул
-- потоков делится между ними — при UNION из двух запросов каждая ветка
-- получает примерно половину. При большом числе подзапросов в UNION ALL
-- все они открывают буферы чтения одновременно, что может приводить к
-- пиковому потреблению памяти пропорционально числу веток.
-- Настройка max_streams_for_union_step, ограничивающая число одновременно
-- активных потоков в UNION-шаге (снижает пиковую память без потери
-- параллелизма в целом), появилась в версии 26.5 — на более ранних версиях
-- число веток UNION ALL приходится ограничивать самостоятельно.
-- Важно: в колоночных СУБД и в BI практиках UNION часто является хорошей
-- альтернативой JOION

-- * Выполните запрос, чтобы найти реально существующие сгенеренные значения
-- в sender_account_id и receiver_account_id

SELECT
    transaction_id,
    sender_account_id,
    receiver_account_id,
    merchant_category,
    country,
    transaction_date,
    amount,
    status
FROM financial_transactions
WHERE (sender_account_id   = 42345 -- подставьте существующее значение
    OR receiver_account_id = 42345)
  AND transaction_date BETWEEN today() - INTERVAL 2 YEAR AND today();
```

> <sub>60 rows in set. Elapsed: 0.046 sec. Processed 1.10 million rows, 16.49 MB (23.80 million rows/s., 358.48 MB/s.)<br>Peak memory usage: 14.85 MiB.</sub>

```sql
-- Проверка через EXPLAIN: оба Skip индекса должны быть в выводе
EXPLAIN indexes = 1
SELECT
    transaction_id,
    sender_account_id,
    receiver_account_id,
    amount,
    status
FROM financial_transactions
WHERE (sender_account_id   = 42345 -- подставьте существующее значение
    OR receiver_account_id = 42345)
  AND transaction_date BETWEEN today() - INTERVAL 2 YEAR AND today();
```

В выводе должна быть секция вида:

> <sub>Skip<br>&nbsp;&nbsp;Name: &lt;Combined skip indexes&gt;<br>&nbsp;&nbsp;Description: Final set of granules after AND/OR processing<br>&nbsp;&nbsp;Parts: 16/16<br>&nbsp;&nbsp;Granules: 29/853</sub>

```sql
-- Замер эффекта: с индексами vs без индексов
SELECT * FROM financial_transactions
WHERE (sender_account_id   = 42345 -- подставьте существующее значение
    OR receiver_account_id = 42345)
  AND transaction_date BETWEEN today() - INTERVAL 2 YEAR AND today()
FORMAT Null
SETTINGS use_query_condition_cache = 0;
```

> <sub>60 rows in set. Elapsed: 0.052 sec. Processed 1.10 million rows, 23.91 MB (21.06 million rows/s., 459.89 MB/s.)<br>Peak memory usage: 24.75 MiB.</sub>

```sql
SELECT * FROM financial_transactions
WHERE (sender_account_id   = 42345 -- подставьте существующее значение
    OR receiver_account_id = 42345)
  AND transaction_date BETWEEN today() - INTERVAL 2 YEAR AND today()
FORMAT Null
SETTINGS use_query_condition_cache = 0, use_skip_indexes = 0;
```

> <sub>64 rows in set. Elapsed: 0.248 sec. Processed 33.52 million rows, 212.40 MB (135.15 million rows/s., 856.44 MB/s.)<br>Peak memory usage: 29.67 MiB.</sub>

Сравнение: без skip-индексов читается почти вся таблица (33.52 млн строк
вместо 1.10 млн с индексами) — почти в 31 раз больше данных, запрос
примерно в 4.8 раза медленнее.

