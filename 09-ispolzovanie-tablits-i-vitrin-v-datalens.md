# Использование таблиц и витрин в Yandex DataLens

В этом рецепте в качестве сквозного примера мы взяли не саму таблицу `financial_transactions`,
а представление `financial_transactions_analytics` из рецепта [Использование таблиц. Представления и оконные функции](07-ispolzovanie-tablits-predstavleniya-i-okonnye-funktsii.md). В нем уже
посчитаны оконные функции — `amount_rank_in_segment`, `risk_percentile_in_country`
и другие. Принцип оптимизации тот же, что и в упомянутом рецепте: BI-система получает уже готовые колонки и ничего не досчитывает сама.

## Практика: датасет и чарт на представлении с оконными функциями

**1. Подключение.** Если подключение к ClickHouse® в Yandex DataLens еще не
создано, создайте его: **Подключения → Создать подключение → ClickHouse®**. Для
Yandex Managed Service for ClickHouse® указываются хост, порт HTTP-интерфейса, пользователь и
пароль. Для BI-системы имеет смысл заводить отдельного пользователя с доступом
только на чтение (см. рецепт [Развертывание ClickHouse®](02-razvertyvanie-clickhouse.md)). Уровень доступа SQL-запросов: **SQL на чтение**.
После проверки соединения нажмите **Создать подключение**.

**2. Создание датасета.** На странице подключения нажмите **Создать датасет**.
В списке таблиц источника выберите не `financial_transactions`, а
`financial_transactions_analytics`. Представления ClickHouse® видны в этом
списке наравне с обычными таблицами, отдельно выбирать режим SQL не
требуется. Переместите таблицу в рабочую область, перейдите на вкладку
**Поля**, назовите и сохраните датасет.

**3. Вычисляемое поле.** На вкладке **Поля** нажмите **Добавить поле**. Пример,
который основан на уже посчитанных в представлении оконных функциях, — метка
риск-сегмента по перцентилю `risk_percentile_in_country`:

```
IF([risk_percentile_in_country] > 0.9, 'Высокий риск (топ 10%)',
   IF([risk_percentile_in_country] > 0.5, 'Средний риск', 'Низкий риск'))
```

Назовите поле, например, `Риск-сегмент` и создайте его.

**4. Чарт.** На странице датасета нажмите **Создать чарт**. Раскладка для
примера: `merchant_category` — в **Измерения** (по оси X), `Риск-сегмент` —
в **Измерения** как разбивку (легенда/цвет), количество транзакций
`count()` по `transaction_id` — в **Показатели** (ось Y). Yandex DataLens при
этом просто группирует и считает `count()` — сам перцентиль он не рассчитывает, вся тяжелая часть уже отработала в ClickHouse® при обращении к
представлению.

## Параметры датасета и параметризованные представления в БД

Yandex DataLens умеет передавать значения параметров датасета в SQL-подзапрос
источника через плейсхолдер `{{имя_параметра}}`. Однако это
работает, только если датасет описан произвольным SQL-запросом, а не
выбором таблицы из списка, как в разделе «Практика: датасет и чарт на представлении с оконными функциями». 

Общий порядок работы:

1. В датасете включить параметризацию, на вкладке **Параметры** добавить
   параметр (название, тип, значение по умолчанию), включить для него
   **Разрешить использовать в настройке источника**.
2. В SQL-запросе источника подставить `{{имя_параметра}}`, например,
   в `WHERE` или, как в этом рецепте, в качестве значения параметра
   параметризованного представления ClickHouse®. В самом ClickHouse® это
   отдельная сущность: `CREATE VIEW ... AS SELECT ... WHERE col = {p:Type}`,
   вызывается как `view_name(p = значение)`.

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


Ниже пример того, как выглядит источник в Yandex DataLens. Кавычки для `country`
нужны явно — `{{...}}`, в Yandex DataLens это текстовая подстановка, а не
параметр с типом:

```sql
SELECT *
FROM financial_transactions_top_by_country(
    country = '{{country_param}}',
    min_risk_percentile = {{min_risk_percentile_param}}
);
```


## Очистка тестовых структур

```sql
-- Шаг 1: Удалить все материализованные представления (зависимые объекты первыми)
DROP VIEW IF EXISTS financial_transactions_daily_summary_mv;
DROP VIEW IF EXISTS financial_transactions_risk_by_country_mv;
DROP VIEW IF EXISTS financial_transactions_risk_snapshots_mv;
DROP VIEW IF EXISTS financial_transactions_status_report_mv;

-- Шаг 2: Удалить таблицы-получатели (target tables)
DROP TABLE IF EXISTS financial_transactions_daily_summary;
DROP TABLE IF EXISTS financial_transactions_risk_by_country;
DROP TABLE IF EXISTS financial_transactions_risk_snapshots;
DROP TABLE IF EXISTS financial_transactions_status_report;
DROP TABLE IF EXISTS financial_transactions_analytics;
DROP TABLE IF EXISTS financial_transactions_top_by_country;

-- Шаг 3: Удалить словари
DROP DICTIONARY IF EXISTS merchant_category_dict;
DROP TABLE IF EXISTS merchant_category_dict_source;

-- Шаг 4: Удалить исходную таблицу (source table) последней
DROP TABLE IF EXISTS financial_transactions;
```

## Итоги

Вы разобрались, как использовать подготовленные в ClickHouse® таблицы и представления в Yandex DataLens. 

Что важно запомнить:

- Представления ClickHouse® можно использовать в Yandex DataLens как обычные таблицы.
- Если производные показатели уже рассчитаны в ClickHouse®, Yandex DataLens использует готовые колонки и не пересчитывает их сам.
- Параметры датасета можно передавать в SQL-запрос источника через `{{имя_параметра}}`, если датасет построен на произвольном SQL-запросе.
- Параметризованные представления ClickHouse® можно вызывать из Yandex DataLens, передавая в них значения параметров датасета.
