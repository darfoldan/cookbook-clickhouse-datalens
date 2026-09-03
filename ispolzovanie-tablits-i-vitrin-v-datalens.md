## Использование таблиц и витрин в Datalens

В качестве сквозного примера возьмем не саму таблицу `financial_transactions`,
а представление `financial_transactions_analytics` из раздела «Представление» — то, в котором уже
посчитаны оконные функции (`amount_rank_in_segment`, `risk_percentile_in_country`
и т.д.). Идея та же, что и в разделе «Использование таблиц - представления и оконные функции»: BI получает уже готовые колонки и не
должен ничего досчитывать сам.

### Практика: датасет и чарт на представлении с оконными функциями

**1. Подключение.** Если подключение к этому ClickHouse® в DataLens еще не
создано — **Подключения → Создать подключение → ClickHouse®**. Для
Managed Service указываются хост, порт HTTP-интерфейса, пользователь и
пароль; для BI имеет смысл заводить отдельного пользователя с доступом
только на чтение (см. 0.1). Уровень доступа SQL-запросов — «SQL на чтение».
После проверки соединения — **Создать подключение**.

**2. Создание датасета.** На странице подключения — **Создать датасет**.
В списке таблиц источника выбираем не `financial_transactions`, а
`financial_transactions_analytics` — представления ClickHouse® видны в этом
списке наравне с обычными таблицами, отдельно выбирать «режим SQL» не
требуется. Перетаскиваем таблицу на рабочую область, переходим на вкладку
**Поля**, сохраняем датасет с названием.

**3. Вычисляемое поле.** На вкладке **Поля** — **Добавить поле**. Пример,
завязанный на уже посчитанные в представлении оконные функции — метка
риск-сегмента по перцентилю `risk_percentile_in_country`:

```
IF([risk_percentile_in_country] > 0.9, 'Высокий риск (топ 10%)',
   IF([risk_percentile_in_country] > 0.5, 'Средний риск', 'Низкий риск'))
```

Называем поле, например, `Риск-сегмент`, и создаем.

**4. Чарт.** На странице датасета — **Создать чарт**. Раскладка для
примера: `merchant_category` — в **Измерения** (по оси X), `Риск-сегмент` —
в **Измерения** как разбивку (легенда/цвет), количество транзакций
(`count()` по `transaction_id`) — в **Показатели** (ось Y). DataLens при
этом просто группирует и считает `count()` — сам перцентиль он не считает
ни разу, вся тяжелая часть уже отработала в ClickHouse® при обращении к
представлению.

### Задел: параметры датасета и параметризированные представления в БД

DataLens умеет передавать значения параметров датасета в SQL-подзапрос
источника через плейсхолдер `{{имя_параметра}}` — но это
работает только когда датасет описан произвольным SQL-запросом, а не
выбором таблицы из списка (как в разделе «Практика: датасет и чарт на представлении с оконными функциями»). Порядок в общих чертах:

1. В датасете включить параметризацию, на вкладке **Параметры** добавить
   параметр (название, тип, значение по умолчанию), включить для него
   **Разрешить использовать в настройке источника**.
2. В SQL-запросе источника подставить `{{имя_параметра}}` — например,
   в `WHERE` или, как в этом рецепте, в качестве значения параметра
   параметризированного представления ClickHouse® (в самом ClickHouse® это
   отдельная сущность: `CREATE VIEW ... AS SELECT ... WHERE col = {p:Type}`,
   вызывается как `view_name(p = значение)`).

Вариант такого представления над `financial_transactions_analytics` — вывод
топа сегмента по стране с фильтром по минимальному риск-перцентилю,
задаваемому параметром:

```sql
CREATE VIEW financial_transactions_top_by_country AS
SELECT
    transaction_id,
    account_id,
    merchant_category,
    country,
    transaction_date,
    amount,
    risk_score,
    amount_rank_in_segment,
    risk_percentile_in_country
FROM financial_transactions_analytics
WHERE country = {country:String}
  AND risk_percentile_in_country >= {min_risk_percentile:Float64}
ORDER BY amount_rank_in_segment;
```

Проверка из `clickhouse-client`:

```sql
SELECT count()
FROM financial_transactions_top_by_country(country = 'US', min_risk_percentile = 0.9);
```

> <sub>1 rows in set. Elapsed: 3.662 sec. Processed 50.00 million rows, 700.02 MB (13.65 million rows/s., 191.16 MB/s.)<br>Peak memory usage: 545.29 MiB.</sub>


В датасете DataLens источник тогда выглядит так (кавычки для `country`
нужны явно — `{{...}}` в DataLens это текстовая подстановка, а не
параметр с типом):

```sql
SELECT *
FROM financial_transactions_top_by_country(
    country = '{{country_param}}',
    min_risk_percentile = {{min_risk_percentile_param}}
);
```


### Очистка тестовых структур

```sql
-- Шаг 1: Удаляем все Materialized Views (зависимые объекты первыми)
DROP VIEW IF EXISTS financial_transactions_daily_summary_mv;
DROP VIEW IF EXISTS financial_transactions_risk_by_country_mv;
DROP VIEW IF EXISTS financial_transactions_risk_snapshots_mv;
DROP VIEW IF EXISTS financial_transactions_status_report_mv;

-- Шаг 2: Удаляем таблицы-получатели (target tables)
DROP TABLE IF EXISTS financial_transactions_daily_summary;
DROP TABLE IF EXISTS financial_transactions_risk_by_country;
DROP TABLE IF EXISTS financial_transactions_risk_snapshots;
DROP TABLE IF EXISTS financial_transactions_status_report;
DROP TABLE IF EXISTS financial_transactions_analytics;
DROP TABLE IF EXISTS financial_transactions_top_by_country;

-- Шаг 3: Удаляем словари
DROP DICTIONARY IF EXISTS merchant_category_dict;
DROP TABLE IF EXISTS merchant_category_dict_source;

-- Шаг 4: Удаляем исходную таблицу (source table) последней
DROP TABLE IF EXISTS financial_transactions;
```
