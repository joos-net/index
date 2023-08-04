# Домашнее задание к занятию "`Индексы`"
# `Островский Евгений`


### Инструкция по выполнению домашнего задания

   1. Сделайте `fork` данного репозитория к себе в Github и переименуйте его по названию или номеру занятия, например, https://github.com/имя-вашего-репозитория/git-hw или  https://github.com/имя-вашего-репозитория/7-1-ansible-hw).
   2. Выполните клонирование данного репозитория к себе на ПК с помощью команды `git clone`.
   3. Выполните домашнее задание и заполните у себя локально этот файл README.md:
      - впишите вверху название занятия и вашу фамилию и имя
      - в каждом задании добавьте решение в требуемом виде (текст/код/скриншоты/ссылка)
      - для корректного добавления скриншотов воспользуйтесь [инструкцией "Как вставить скриншот в шаблон с решением](https://github.com/netology-code/sys-pattern-homework/blob/main/screen-instruction.md)
      - при оформлении используйте возможности языка разметки md (коротко об этом можно посмотреть в [инструкции  по MarkDown](https://github.com/netology-code/sys-pattern-homework/blob/main/md-instruction.md))
   4. После завершения работы над домашним заданием сделайте коммит (`git commit -m "comment"`) и отправьте его на Github (`git push origin`);
   5. Для проверки домашнего задания преподавателем в личном кабинете прикрепите и отправьте ссылку на решение в виде md-файла в вашем Github.
   6. Любые вопросы по выполнению заданий спрашивайте в чате учебной группы и/или в разделе “Вопросы по заданию” в личном кабинете.
   
Желаем успехов в выполнении домашнего задания!
   
### Дополнительные материалы, которые могут быть полезны для выполнения задания

1. [Руководство по оформлению Markdown файлов](https://gist.github.com/Jekins/2bf2d0638163f1294637#Code)

---

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

Для таблицы
```SQL
SELECT table_name, CONCAT(ROUND(INDEX_LENGTH /DATA_LENGTH*100), ' %') AS percent
FROM INFORMATION_SCHEMA.TABLES
WHERE table_name = "city";
```
![1](https://github.com/joos-net/index/blob/main/1.png)

Для всех таблиц
```SQL
SELECT CONCAT(ROUND(SUM(INDEX_LENGTH) / SUM(DATA_LENGTH) * 100), ' %') AS percent
FROM INFORMATION_SCHEMA.TABLES;
```
![3](https://github.com/joos-net/index/blob/main/3.png)

## Исправления
```SQL
SELECT CONCAT(ROUND(SUM(INDEX_LENGTH) / (SUM(INDEX_LENGTH)+SUM(DATA_LENGTH)) * 100), ' %') AS percent
FROM INFORMATION_SCHEMA.TABLES;
```
![4](https://github.com/joos-net/index/blob/main/4.png)

---

### Задание 2

Выполните explain analyze следующего запроса:

```SQL
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

```SQL
-> Limit: 400 row(s)  (cost=0..0 rows=0) (actual time=7312..7312 rows=391 loops=1)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=7312..7312 rows=391 loops=1)
        -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=7312..7312 rows=391 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=3014..7033 rows=642000 loops=1)
                -> Sort: c.customer_id, f.title  (actual time=3014..3139 rows=642000 loops=1)
                    -> Stream results  (cost=21.2e+6 rows=16e+6) (actual time=0.626..2102 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=21.2e+6 rows=16e+6) (actual time=0.619..1743 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=19.6e+6 rows=16e+6) (actual time=0.614..1521 rows=642000 loops=1)
                                -> Nested loop inner join  (cost=18e+6 rows=16e+6) (actual time=0.607..1253 rows=642000 loops=1)
                                    -> Inner hash join (no condition)  (cost=1.58e+6 rows=15.8e+6) (actual time=0.589..68.4 rows=634000 loops=1)
                                        -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.65 rows=15813) (actual time=0.0555..8.8 rows=634 loops=1)
                                            -> Table scan on p  (cost=1.65 rows=15813) (actual time=0.0404..5.82 rows=16044 loops=1)
                                        -> Hash
                                            -> Covering index scan on f using idx_title  (cost=111 rows=1000) (actual time=0.0579..0.392 rows=1000 loops=1)
                                    -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.938 rows=1.01) (actual time=0.00114..0.00165 rows=1.01 loops=634000)
                                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=250e-6 rows=1) (actual time=191e-6..220e-6 rows=1 loops=642000)
                            -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=250e-6 rows=1) (actual time=149e-6..178e-6 rows=1 loops=642000)
```
В выводе видим что мы сортируем по 2-м большим таблицам и проходим по большому количеству строк. Так как дополнительная сортировка по названию фильмов нигде больше не используется, то уберем ее и таблицу film. Еще отсутствует необходимость в таблице inventory и проверке условия inventory_id
```SQL
EXPLAIN ANALYZE select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id)
from payment p, rental r, customer c
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id
```
```SQL
-> Limit: 400 row(s)  (cost=0..0 rows=0) (actual time=14.9..15.1 rows=391 loops=1)
    -> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=14.9..15 rows=391 loops=1)
        -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=14.9..14.9 rows=391 loops=1)
            -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id )   (actual time=13.2..14.6 rows=642 loops=1)
                -> Sort: c.customer_id  (actual time=13.1..13.2 rows=642 loops=1)
                    -> Stream results  (cost=23632 rows=16003) (actual time=0.127..12.9 rows=642 loops=1)
                        -> Nested loop inner join  (cost=23632 rows=16003) (actual time=0.12..12.4 rows=642 loops=1)
                            -> Nested loop inner join  (cost=18031 rows=16003) (actual time=0.11..11.4 rows=642 loops=1)
                                -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1606 rows=15813) (actual time=0.0887..9.33 rows=634 loops=1)
                                    -> Table scan on p  (cost=1606 rows=15813) (actual time=0.0688..6.96 rows=16044 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.938 rows=1.01) (actual time=0.0021..0.00286 rows=1.01 loops=634)
                            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.00126..0.0013 rows=1 loops=642)
```
Основное назначение OVER PARTITION BY - это группировка по значению для sum, такая группировка оставляет все строки, GROUP BY сокращает количество строк в запросе с помощью их группировки, попробуем использовать GROUP BY
```SQL
EXPLAIN ANALYZE select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount)
from payment p, rental r, customer c
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id
GROUP BY c.customer_id;
```
```SQL
-> Limit: 400 row(s)  (actual time=12.1..12.1 rows=391 loops=1)
    -> Sort with duplicate removal: `concat(c.last_name, ' ', c.first_name)`, `sum(p.amount)`  (actual time=12.1..12.1 rows=391 loops=1)
        -> Table scan on <temporary>  (actual time=11.8..11.8 rows=391 loops=1)
            -> Aggregate using temporary table  (actual time=11.8..11.8 rows=391 loops=1)
                -> Nested loop inner join  (cost=23632 rows=16003) (actual time=0.0901..10.7 rows=642 loops=1)
                    -> Nested loop inner join  (cost=18031 rows=16003) (actual time=0.0839..9.72 rows=642 loops=1)
                        -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1606 rows=15813) (actual time=0.0682..7.8 rows=634 loops=1)
                            -> Table scan on p  (cost=1606 rows=15813) (actual time=0.0551..5.83 rows=16044 loops=1)
                        -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.938 rows=1.01) (actual time=0.00198..0.00275 rows=1.01 loops=634)
                    -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.00123..0.00127 rows=1 loops=642)
```
Получаем запрос
```SQL
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount)
from payment p, rental r, customer c
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id
GROUP BY c.customer_id;
```
![2](https://github.com/joos-net/index/blob/main/2.png)

## Исправления
```SQL
CREATE INDEX inx_pay_date ON payment(payment_date);

EXPLAIN ANALYZE select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount)
from payment p, rental r, customer c
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and c.customer_id = r.customer_id 
GROUP BY c.customer_id;
```
```SQL
-> Limit: 400 row(s)  (actual time=67..67.1 rows=391 loops=1)
    -> Sort with duplicate removal: `concat(c.last_name, ' ', c.first_name)`, `sum(p.amount)`  (actual time=67..67 rows=391 loops=1)
        -> Table scan on <temporary>  (actual time=66.7..66.8 rows=391 loops=1)
            -> Aggregate using temporary table  (actual time=66.7..66.7 rows=391 loops=1)
                -> Nested loop inner join  (cost=11265 rows=16005) (actual time=0.513..65.2 rows=642 loops=1)
                    -> Nested loop inner join  (cost=5663 rows=16005) (actual time=0.248..26 rows=16044 loops=1)
                        -> Table scan on c  (cost=61.2 rows=599) (actual time=0.143..0.6 rows=599 loops=1)
                        -> Index lookup on r using idx_fk_customer_id (customer_id=c.customer_id)  (cost=6.68 rows=26.7) (actual time=0.0338..0.0402 rows=26.8 loops=599)
                    -> Index lookup on p using inx_pay_date (payment_date=r.rental_date), with index condition: (cast(p.payment_date as date) = '2005-07-30')  (cost=0.25 rows=1) (actual time=0.00225..0.00228 rows=0.04 loops=16044)
```
Без индексов
![101](https://github.com/joos-net/index/blob/main/101.png)

С индексом payment_date
![102](https://github.com/joos-net/index/blob/main/102.png)

---
## Дополнительные задания (со звездочкой*)

Эти задания дополнительные (не обязательные к выполнению) и никак не повлияют на получение вами зачета по этому домашнему заданию. Вы можете их выполнить, если хотите глубже и/или шире разобраться в материале.

### Задание 3

Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.

Приведите ответ в свободной форме.

В PostgreSQL используются следующие типы индексов - B-Tree, Hash, GiST, SP-GiST, GIN и BRIN
- B-TREE — древовидная структура данных, популярная для использования в индексах БД
- HASH-индексы предполагают хранение не самих значений, а их хэшей, благодаря чему уменьшается размер и увеличивается скорость обработки индексов из больших полей.
- GiST - Generalized Search Tree, Обобщенное поисковое дерево — структура индекса, которая является обобщенной разновидностью R-tree. GiST представляет собой сбалансированное (по высоте) дерево, концевые узлы которого содержат пары (key, rid), где key — ключ, а rid — указатель на соответствующую запись на странице данных
- SP-GiST - Space-Partitioned GiST (GiST с разбиением пространства). SP-GiST поддерживает деревья поиска с разбиением, что облегчает разработку широкого спектра различных несбалансированных структур данных.
- GIN - Generalized INverted index — реализация обратного индекса, используемая в СУБД PostgreSQL, в частности, для полнотекстового поиска и поиска по содержимому полей типа JSON
- BRIN - Block Range Index — техника индексации данных, предназначенная для обработки больших таблиц, в которых значение индексируемого столбца имеет некоторую естественную корреляцию с физическим положением строки в таблице

В MySQL
- B-TREE
- R-TREE
- INVERTED
- HASH

В MySQL не используются следующие типы индексов - SP-GiST и BRIN
