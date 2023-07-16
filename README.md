# Домашнее задание к занятию `"«Индексы»"` - `Дьяконов Алексей`


##### Задание 1. Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

```
    SELECT  SUM(index_length)/SUM(data_length)*100 AS otnoshenie
    FROM INFORMATION_SCHEMA.TABLES
    WHERE table_schema = "sakila";
```

![zadanie_1](/img/1_1.jpg)

##### Задание 2. Выполните explain analyze запроса:

```
    EXPLAIN ANALYZE
    SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name), SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title)
    FROM payment p, rental r, customer c, inventory i, film f
    WHERE DATE(p.payment_date) = '2005-07-30' AND p.payment_date = r.rental_date AND r.customer_id = c.customer_id AND i.inventory_id = r.inventory_id;

```
`Результат выполнения`
![zadanie_2_1](/img/2_1.jpg)

`Можно обратить внимание, что основная проблема медленного выполнения в inner hash join (no condition) возникающего из за оконной функции sum(p.amount) over (partition by c.customer_id, f.title). К сожалению информации о том, поддерживают ли оконные функции парционирование по нескольким столбцам из разных  таблиц. Поэтому сначала я решил применить inner join в запросе, но не учел, что в таком случае не будет работать distinct. После этого я решил посмотреть в сторогу индексов по этим столбцам, но они уже были созданы. В итоге, внимательно посмотрев на функцию , попробовал удалить f.title и film f из запроса. Вывод запроса не изменился,  а скорость исполнения выросла. Добавив индекс по paiment_date получил следующий итог:  `
```
    CREATE INDEX payment_date ON payment(payment_date);

    EXPLAIN ANALYZE
    SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name), SUM(p.amount) OVER (PARTITION BY c.customer_id)
    FROM payment p, rental r, customer c, inventory i
    WHERE DATE(p.payment_date) = '2005-07-30' AND p.payment_date = r.rental_date AND r.customer_id = c.customer_id AND i.inventory_id = r.inventory_id;
```

`Результат выполнения`
![zadanie_2_2](/img/2_2.jpg)

`Скорость выполнения запроса выросла, inner hash join (no condition) ушёл`

##### Задание 3.

`В  PostgreSQL используются следующие индексы:`

1. `B-tree используется в MySQL - могут использоваться с  <   <=   =   >=   > `
2. `HASH - используется в MySQL - хранят хеш-код, поэтому используются с =`
3. `GiST -представляют собой инфраструктуру, позволяющую реализовать много разных стратегий индексирования. Как следствие, GiST-индексы могут применяться с разными операторами, в зависимости от стратегии индексирования (класса операторов)<<   &<   &>   >>   <<|   &<|   |&>   |>>   @>   <@   ~=   &&  `
4. `SP-GiST  - предоставляют инфраструктуру, поддерживающую различные типы поиска. SP-GiST позволяет организовывать на диске самые разные несбалансированные структуры данных. Используются с операторами <<   >>   ~=   <@   <<|   |>>`
5. `GIN -  представляют собой «инвертированные индексы», в которых могут содержаться значения с несколькими ключами, например массивы. Используются с операторами - <@   @>   =   && `
6. `BRIN -  (сокращение от Block Range INdexes, Индексы зон блоков) хранят обобщённые сведения о значениях, находящихся в физически последовательно расположенных блоках таблицы. Используется с <   <=   =   >=   > `