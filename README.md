# Домашнее задание к занятию "`Индексы`" - `Евгений Головенко`

---

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

### Решение 1

```
select  
    ROUND(
    	SUM(INDEX_LENGTH ) / SUM(DATA_LENGTH) * 100,
        2
    ) AS 'процент индексов от размера таблиц'
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'sakila'
  AND DATA_LENGTH > 0;
```

Результат запроса:

```
процент индексов от размера таблиц|
----------------------------------+
                             54.68|
```

<img width="463" height="364" alt="sakila_12-5-1" src="https://github.com/user-attachments/assets/0826937d-2148-419e-8ca4-3d2efb48690d" />

---

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

### Решение 2

```sql
explain analyze
select distinct 
	concat(c.last_name, ' ', c.first_name),
	sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30'
	and p.payment_date = r.rental_date
	and r.customer_id = c.customer_id
	and i.inventory_id = r.inventory_id;
```
Получаем:
```sql
EXPLAIN                                                                                                                                                                                                                                                        |
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=4306..4306 rows=391 loops=1)¶    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=4306..4306 rows=391 loops=1)¶        -> Window aggregate with buffering: sum(p.amount|
```

Эммм... сделаем красиво:

```sql
-> Table scan on <temporary>
   (cost=2.5..2.5 rows=0)
   (actual time=4306..4306 rows=391 ...)

    -> Temporary table with deduplication
       (cost=0..0 rows=0)
       (actual time=4306..4306 rows=391 ...)

        -> Window aggregate with buffering: sum(p.amount)
```

Что этот запрос выполняет:
- выбирает платежи за 30 июля 2005
- связывает `payment` -> `rental` -> `customer` -> `inventory`, но не трогает `film`
- считает сумму платежей по покупателю и фильму (`partition by c.customer_id, f.title`) и использует оконную функцию `over (...)`, т.е. строки не склеиваются в одну, каждая строка остается в результате, но рядом с ней появляется вычисленное значение.
- убирает дубликаты через DISTINCT (следствие отсутствия связи с `film`).


MySQL создал временную таблицу (`Table scan on <temporary>`), записал туда промежуточные результаты и затем полностью её просканировал. Потом удалял дублирующиеся строки (`Temporary table with deduplication`).

Что тут не так:

Таблицы соединялись не совсем явно. Отдельный вопрос к таблице `film`. Нет условия `i.film_id = f.film_id` в результате каждая строка из уже объединенных таблиц соединяется с каждой строкой таблицы `film`, что увеличивает объем обрабатываемых данных.

Основное узкое место это создание временной таблицы и работа с ней (запись данных, сортировка, удаление дубликатов, полный проход по таблице).

Суммирование `sum(p.amount) over (partition by c.customer_id, f.title)` осуществляется не в процессе, а только после накопления данных.

Использование `date(p.payment_date)` в условии `WHERE`, т.е. для каждой строки вызывается функция `date`, из-за этого индекс по `payment_date` не используется и MySQL вынужден просматривать всю таблицу `payment`.

Как вот это можно оптимизировать:

- использовать явные JOIN,
- таблицы соединить корректно,
- фильтр по дате с использованием диапазона, а не функции,
- вместо оконной функции использовать GROUP BY.

Оптимизированный запрос может выглядеть как-то так:

```sql
SELECT
    CONCAT(c.last_name, ' ', c.first_name),
    f.title,
    SUM(p.amount)
FROM payment p
JOIN rental r ON p.rental_id = r.rental_id
JOIN customer c ON r.customer_id = c.customer_id
JOIN inventory i ON r.inventory_id = i.inventory_id
JOIN film f ON i.film_id = f.film_id
WHERE p.payment_date >= '2005-07-30'
  AND p.payment_date <  '2005-07-31'
GROUP BY c.customer_id, f.film_id;
```

Что же покажет `explain analyze`?

```sql
-> Table scan on <temporary>
   (actual time=21.7..21.9 rows=634...)
   -> Aggregate using temporary table
      (actual time=21.7..21.7 rows=634...)
      -> Nested loop inner join  (cost=5527 rows=1833)
         (actual time=0.456..19.9 rows=634...)
```

Чтож, вполне приемлемо.

Строка `-> Nested loop inner join` указывает на то, что база данных использует алгоритм вложенных циклов. Это значит, что MySQL берет строку из одной таблицы (ведущей) и ищет соответствующие строки в другой таблице. Этот алгоритм эффективен только тогда, когда поля, по которым идет соединение `(on ...)`, проиндексированы. Если бы индексов на внешних ключах (rental_id, inventory_id, film_id, customer_id) не было, база данных, скорее всего, выбрала бы Hash Join или работала бы значительно медленнее.

Индекс для фильтрации `WHERE` возможно используется. Общее время выполнения (около 21 мс) и количество строк (634) для такого количества джойнов подсказывают, что индекс по полю payment_date в таблице `payment` скорее всего используется. Если бы индекса не было, базе пришлось бы делать Full Table Scan всей таблицы `payment`, что заняло бы больше времени, что было видно до оптимизации.

Индекс для группировки `GROUP BY` не используется.

```sql
-> Table scan on <temporary>
   -> Aggregate using temporary table
```

Запрос группирует данные по полям из двух разных таблиц `customer` и `film`, которые находятся на разных концах цепочки джойнов. Существование одного индекса, который покрыл бы эту группировку, было бы весьма затруднительным. Поэтому здесь используется временная таблица.

---
