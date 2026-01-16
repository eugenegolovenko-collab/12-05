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
	concat(c.last_name, ' ', c.first_name) as full_name,
	sum(p.amount) as total_amount_per_customer_per_film
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30'
	and p.payment_date = r.rental_date
	and r.customer_id = c.customer_id
	and i.inventory_id = r.inventory_id;
```
Получаем:
```sql
-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=4235..4235 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=4235..4235 rows=391 loops=1)
        -> Window aggregate with buffering: sum(p.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=2474..4083 rows=642000 loops=1)
            -> Sort: c.customer_id, f.title  (actual time=2474..2533 rows=642000 loops=1)
                -> Stream results  (cost=22.7e+6 rows=16.7e+6) (actual time=0.634..1894 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=22.7e+6 rows=16.7e+6) (actual time=0.63..1647 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=21e+6 rows=16.7e+6) (actual time=0.624..1480 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=19.3e+6 rows=16.7e+6) (actual time=0.617..1302 rows=642000 loops=1)
                                -> Inner hash join (no condition)  (cost=1.65e+6 rows=16.5e+6) (actual time=0.605..42.2 rows=634000 loops=1)
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.72 rows=16500) (actual time=0.286..5.56 rows=634 loops=1)
                                        -> Table scan on p  (cost=1.72 rows=16500) (actual time=0.276..4.11 rows=16044 loops=1)
                                    -> Hash
                                        -> Covering index scan on f using idx_title  (cost=103 rows=1000) (actual time=0.0336..0.24 rows=1000 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date = p.payment_date)  (cost=0.969 rows=1.01) (actual time=0.00141..0.00188 rows=1.01 loops=634000)
                            -> Single-row index lookup on c using PRIMARY (customer_id = r.customer_id)  (cost=250e-6 rows=1) (actual time=156e-6..172e-6 rows=1 loops=642000)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id = r.inventory_id)  (cost=250e-6 rows=1) (actual time=138e-6..154e-6 rows=1 loops=642000)
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
select
	concat(c.last_name, ' ', c.first_name) as full_name,
	sum(p.amount) as total_amount_per_customer_per_film
from payment p 
join
	rental r on p.payment_date = r.rental_date 
join
	customer c on r.customer_id = c.customer_id 
join
	inventory i on r.inventory_id = i.inventory_id 
join
	film f on i.film_id = f.film_id 
where
	p.payment_date >= '2005-07-30' and p.payment_date < '2005-07-31'
group by 	c.customer_id, c.last_name, c.first_name;
```

Что же покажет `explain analyze`?

```sql
-> Table scan on <temporary>  (actual time=11.1..11.1 rows=391 loops=1)
    -> Aggregate using temporary table  (actual time=11.1..11.1 rows=391 loops=1)
        -> Nested loop inner join  (cost=5584 rows=1855) (actual time=0.338..10.1 rows=642 loops=1)
            -> Nested loop inner join  (cost=4934 rows=1855) (actual time=0.331..8.86 rows=642 loops=1)
                -> Nested loop inner join  (cost=4285 rows=1855) (actual time=0.325..7.64 rows=642 loops=1)
                    -> Nested loop inner join  (cost=3636 rows=1855) (actual time=0.317..6.83 rows=642 loops=1)
                        -> Filter: ((p.payment_date >= TIMESTAMP'2005-07-30 00:00:00') and (p.payment_date < TIMESTAMP'2005-07-31 00:00:00'))  (cost=1674 rows=1833) (actual time=0.306..5.25 rows=634 loops=1)
                            -> Table scan on p  (cost=1674 rows=16500) (actual time=0.296..4.2 rows=16044 loops=1)
                        -> Covering index lookup on r using rental_date (rental_date = p.payment_date)  (cost=0.969 rows=1.01) (actual time=0.00172..0.00235 rows=1.01 loops=634)
                    -> Single-row index lookup on c using PRIMARY (customer_id = r.customer_id)  (cost=0.25 rows=1) (actual time=0.00114..0.00116 rows=1 loops=642)
                -> Single-row index lookup on i using PRIMARY (inventory_id = r.inventory_id)  (cost=0.25 rows=1) (actual time=0.00177..0.00179 rows=1 loops=642)
            -> Single-row covering index lookup on f using PRIMARY (film_id = i.film_id)  (cost=0.25 rows=1) (actual time=0.00176..0.00178 rows=1 loops=642)
```
---
