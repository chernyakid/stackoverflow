# Анализ постов StackOverflow. Часть 2.

**Задания:** [№1](#задание-1), [№2](#задание-2), [№3](#задание-3), [№4](#задание-4), [№5](#задание-5), 
[№6](#задание-6), [№7](#задание-7)

## Задание №1 
Выведите общую сумму просмотров у постов, опубликованных в каждый месяц 2008 года. 
Если данных за какой-либо месяц в базе нет, такой месяц можно пропустить. 
Результат отсортируйте по убыванию общего количества просмотров.

```sql
select sum(views_count) as views_sum,
       date_trunc('month', creation_date)::date as creation_month
from stackoverflow.posts
where creation_date::date between '2008-01-01' and '2008-12-31'
group by date_trunc('month', creation_date)
order by sum(views_count) desc
```
```text
| views_sum | creation_month|
|-----------|---------------| 
| 452928568 | 2008-09-01    | 
| 365400138 | 2008-10-01    | 
| 221759651 | 2008-11-01    | 
| 197792841 | 2008-12-01    | 
| 131367083 | 2008-08-01    | 
| 669895    | 2008-07-01    | 
```


## Задание №2
Выведите имена самых активных пользователей, которые в первый месяц 
после регистрации (включая день регистрации) дали больше 100 ответов. 
Вопросы, которые задавали пользователи, не учитывайте. 
Для каждого имени пользователя выведите количество уникальных значений `user_id`.
Отсортируйте результат по полю с именами в лексикографическом порядке.

```sql
select 
  u.display_name,
  count(distinct p.user_id)
from stackoverflow.users u
join stackoverflow.posts p on u.id = p.user_id
join stackoverflow.post_types pt on pt.id = p.post_type_id
where 
  pt.type like 'Answer' and 
  p.creation_date::date between 
    u.creation_date::date and 
	(u.creation_date::date + interval '1 month') 
group by u.display_name
having count(p.id) > 100
order by u.display_name
```
```text
| display_name     | count |
|------------------|-------|
| 1800 INFORMATION | 1     |
| Adam Bellaire    | 1     |
| Adam Davis       | 1     |
| Adam Liss        | 1     |
...
```


## Задание №3
Выведите количество постов за 2008 год по месяцам. 
Отберите посты от пользователей, которые зарегистрировались в сентябре 2008 года 
и сделали хотя бы один пост в декабре того же года. 
Отсортируйте таблицу по значению месяца по убыванию.

```sql
with post_users as (
  select p.user_id
  from stackoverflow.posts p
  join stackoverflow.users u on p.user_id = u.id
  where 
    u.creation_date::date between '2008-09-01' and '2008-09-30' and
	p.creation_date::date between '2008-12-01' and '2008-12-31'
  group by p.user_id
)
select 
  date_trunc('month', po.creation_date)::date as posts_month,
  count(po.id) as posts_cnt
from stackoverflow.posts po
join post_users pu on pu.user_id = po.user_id
group by date_trunc('month', po.creation_date)
order by date_trunc('month', po.creation_date) desc
```
```text
| posts_month | posts_cnt |
|-------------|-----------|
| 2008-12-01  | 17641     |
| 2008-11-01  | 18294     |
| 2008-10-01  | 27171     |
| 2008-09-01  | 24870     |
| 2008-08-01  | 32        |
```


## Задание №4
Используя данные о постах, выведите несколько полей:
- идентификатор пользователя, который написал пост;
- дата создания поста;
- количество просмотров у текущего поста;
- сумма просмотров постов автора с накоплением.
Данные в таблице должны быть отсортированы по возрастанию идентификаторов пользователей, 
а данные об одном и том же пользователе — по возрастанию даты создания поста.

```sql
select 
  pos.user_id,
  pos.creation_date,
  pos.views_count,
  sum(views_count) over(partition by user_id ORDER BY creation_date) 
from stackoverflow.posts as pos
order by pos.user_id 
```
```text
| user_id | creation_date       | views_count | sum    |
|---------|---------------------|-------------|--------|
| 1       | 2008-07-31 23:41:00 | 480476      | 480476 |
| 1       | 2008-07-31 23:55:38 | 136033      | 616509 |
| 1       | 2008-07-31 23:56:41 | 0           | 616509 |
| 1       | 2008-08-04 02:45:08 | 0           | 616509 |
| 1       | 2008-08-04 04:31:03 | 0           | 616509 |
...
```


## Задание №5
Сколько в среднем дней в период с 1 по 7 декабря 2008 года 
включительно пользователи взаимодействовали с платформой? 
Для каждого пользователя отберите дни, в которые он или она опубликовали хотя бы один пост. 
Нужно получить одно целое число — не забудьте округлить результат.

```sql
with day_posts as (
    select 
	  user_id,
      count(distinct creation_date::date) as day_cnt
    from stackoverflow.posts 
    where creation_date::date between '2008-12-01' and '2008-12-07'
    group by user_id)
select round(avg(day_cnt))
from day_posts

-- 2
```


## Задание №6
На сколько процентов менялось количество постов ежемесячно с 1 сентября по 31 декабря 2008 года? 
Отобразите таблицу со следующими полями:
- Номер месяца.
- Количество постов за месяц.
- Процент, который показывает, насколько изменилось количество постов в текущем месяце по сравнению с предыдущим.

Если постов стало меньше, значение процента должно быть отрицательным, если больше — положительным. 
Округлите значение процента до двух знаков после запятой.

Напомним, что при делении одного целого числа на другое в PostgreSQL в результате получится целое число, 
округлённое до ближайшего целого вниз. 
Чтобы этого избежать, переведите делимое в тип `numeric`.

```sql
with pc as (
  select 
    count(distinct id) count_posts,
    extract(month from creation_date::date) as month_creation  
  from stackoverflow.posts
  where creation_date::date between '2008-09-01' and '2008-12-31'
  group by extract(month from creation_date::date))
select 
  month_creation,
  count_posts::numeric,
  round(((count_posts::numeric / lag(count_posts) over(order by month_creation)) - 1) * 100, 2) as pers
from pc
```
```text
| month_creation | count_posts | pers   |
|----------------|-------------|--------|
| 9              | 70371       |        |
| 10             | 63102       | -10.33 |
| 11             | 46975       | -25.56 |
| 12             | 44592       | -5.07  |

```


## Задание №7
Найдите пользователя, который опубликовал больше всего постов за всё время с момента регистрации. 
Выведите данные его активности за октябрь 2008 года в таком виде:
- номер недели;
- дата и время последнего поста, опубликованного на этой неделе.

```sql
with user_post as (
  select user_id, count(distinct id) as cnt
  from stackoverflow.posts
  group by user_id
  order by cnt desc
  limit 1
),
dtt as (
  select 
    p.user_id,
    p.creation_date,
    extract('week' from p.creation_date) as week_number
  from stackoverflow.posts as p
  join user_post on user_post.user_id = p.user_id
  where date_trunc('month', p.creation_date)::date = '2008-10-01'
)
select 
  distinct week_number::numeric,
  max(creation_date) over (partition by week_number) as post_dt
from dtt
order by week_number;
```
```text
| week_number | post_dt             |
|-------------|---------------------|
| 40          | 2008-10-05 09:00:58 |
| 41          | 2008-10-12 21:22:23 |
| 42          | 2008-10-19 06:49:30 |
| 43          | 2008-10-26 21:44:36 |
| 44          | 2008-10-31 22:16:01 |
```
