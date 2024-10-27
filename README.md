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

```SQL
SELECT CONCAT(ROUND(SUM(INDEX_LENGTH) / (SUM(INDEX_LENGTH)+SUM(DATA_LENGTH)) * 100), ' %') AS percent
FROM INFORMATION_SCHEMA.TABLES;
```
![alt_text](https://github.com/alexeymanzhos/index/blob/main/img/1.jpg)

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
Также стоит отметить, что лучше использовать GROUP BY вместо OVER PARTITION BY.
Получаем запрос

```SQL
CREATE INDEX inx_pay_date ON payment(payment_date);

EXPLAIN ANALYZE select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount)
from payment p, customer c
where payment_date >= '2005-07-30' and payment_date < '2005-07-31' and p.customer_id = c.customer_id 
GROUP BY p.customer_id;
```
```SQL
-> Limit: 400 row(s)  (actual time=4.11..4.2 rows=391 loops=1)
    -> Sort with duplicate removal: `concat(c.last_name, ' ', c.first_name)`, `sum(p.amount)`  (actual time=4.11..4.16 rows=391 loops=1)
        -> Table scan on <temporary>  (actual time=3.72..3.8 rows=391 loops=1)
            -> Aggregate using temporary table  (actual time=3.72..3.72 rows=391 loops=1)
                -> Nested loop inner join  (cost=507 rows=634) (actual time=0.05..2.91 rows=634 loops=1)
                    -> Index range scan on p using inx_pay_date over ('2005-07-30 00:00:00' <= payment_date < '2005-07-31 00:00:00'), with index condition: ((p.payment_date >= TIMESTAMP'2005-07-30 00:00:00') and (p.payment_date < TIMESTAMP'2005-07-31 00:00:00'))  (cost=286 rows=634) (actual time=0.0385..1.7 rows=634 loops=1)
                    -> Single-row index lookup on c using PRIMARY (customer_id=p.customer_id)  (cost=0.25 rows=1) (actual time=0.00163..0.00167 rows=1 loops=634)
```
![alt_text](https://github.com/alexeymanzhos/index/blob/main/img/2.jpg)

