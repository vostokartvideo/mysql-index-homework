# MySQL — индексы 
# Антон Сап

## Задание 1. 

Рассчитал отношение размера вторичных индексов к размеру данных таблиц базы `sakila`.

```sql
SELECT
    SUM(DATA_LENGTH) AS total_data,
    SUM(INDEX_LENGTH) AS total_indexes,
    ROUND(SUM(INDEX_LENGTH) / SUM(DATA_LENGTH) * 100, 2) AS index_percent
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'sakila'
  AND TABLE_TYPE = 'BASE TABLE';
```

**Результат:** данные — **4 374 528 байт**, вторичные индексы — **2 392 064 байт**, отношение — **54,68%**.

Скриншоты:

- [Размеры отдельных таблиц](task1-table-sizes.jpg)
- [Общие размеры данных и индексов](task1-total-sizes.jpg)
- [Итоговый расчёт процента](task1-index-percent.jpg)

## Задание 2.

Проверил исходный запрос с помощью `EXPLAIN`, нашёл проблемы в соединениях и фильтрации, создал индекс и сравнил планы выполнения.

**Выявленные проблемы:** отсутствовала связь `film` с `inventory`; `payment` соединялась с `rental` по дате вместо `rental_id`; `DATE(payment_date)` затрудняла диапазонный поиск по индексу; сочетание оконной функции и `DISTINCT` требовало дополнительных операций.

Создан индекс:

```sql
CREATE INDEX idx_payment_date ON payment (payment_date);
```

Оптимизированный запрос для расчёта суммы платежей по каждому клиенту и фильму за 30 июля 2005 года:

```sql
SELECT
    c.customer_id,
    CONCAT(c.last_name, ' ', c.first_name) AS customer_name,
    f.title AS film_title,
    SUM(p.amount) AS total_amount
FROM payment p
JOIN rental r ON p.rental_id = r.rental_id
JOIN customer c ON r.customer_id = c.customer_id
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN film f ON i.film_id = f.film_id
WHERE p.payment_date >= '2005-07-30'
  AND p.payment_date < '2005-07-31'
GROUP BY c.customer_id, f.title;
```

**Результаты анализа:**

- Первоначально `payment` читалась полным сканированием (`type = ALL`), а `film` не имела условия соединения.
- После исправления связей и добавления индекса MySQL использует `idx_payment_date` (`range`) и первичные ключи при соединениях (`eq_ref`).
- В отдельных запусках `EXPLAIN ANALYZE` вариант с оконной функцией и `DISTINCT` выполнялся примерно за **24 мс**, вариант с `GROUP BY` — примерно за **8 мс**.

Скриншоты этапов:

- [Исходный EXPLAIN](task2-explain-1.jpg)
- [EXPLAIN после исправления связи с film](task2-explain-2-fixed.jpg)
- [Промежуточный EXPLAIN](task2-explain-3.jpg)
- [EXPLAIN после создания индекса](task2-explain-4.jpg)
- [EXPLAIN ANALYZE с оконной функцией](task2-explain-5-analyze.jpg)
- [EXPLAIN ANALYZE с GROUP BY](task2-explain-6-analyze.jpg)
- [Итоговая выборка](task2-explain-8-result.jpg)