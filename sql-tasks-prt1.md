# Анализ постов StackOverflow. Часть 1.

**Задания:** [№1](#задание-1), [№2](#задание-2), [№3](#задание-3), [№4](#задание-4), [№5](#задание-5), 
[№6](#задание-6), [№7](#задание-7), [№8](#задание-8), [№9](#задание-9), [№10](#задание-10), [№11](#задание-11),
[№12](#задание-12), [№13](#задание-13)

## Задание №1 
Найдите количество вопросов, которые набрали больше 300 очков или как минимум 100 раз были добавлены в «Закладки».

```sql
select count(id)
from stackoverflow.posts
where 
  (score > 300 or favorites_count >= 100)
  and post_type_id = 1;

-- 1355
```


## Задание №2
Сколько в среднем в день задавали вопросов с 1 по 18 ноября 2008 включительно? 
Результат округлите до целого числа.

```sql
with id_counts as (
    select 
		count(id) as id,
        date_trunc('day', creation_date)::date as creation_day
    from stackoverflow.posts
    where 
		post_type_id = 1
		and date_trunc('day', creation_date) between '2008-11-01' and '2008-11-18' 
    group by date_trunc('day', creation_date)::date
)
select round(avg(id), 0)
from id_counts;

-- 383
```


## Задание №3
Сколько пользователей получили значки сразу в день регистрации? 
Выведите количество уникальных пользователей.

```sql
select count(distinct u.id) as users_cnt
from stackoverflow.users u 
join stackoverflow.badges b on b.user_id = u.id 
where u.creation_date::date = b.creation_date::date;

-- 7047
```


## Задание №4
Сколько уникальных постов пользователя с именем `Joel Coehoorn` получили хотя бы один голос?

```sql
select count(cv.id)
from (
	select ps.id
	from stackoverflow.posts as ps
	join stackoverflow.votes as v on ps.id = v.post_id
	join stackoverflow.users as u on ps.user_id = u.id
	where 
		u.display_name like 'Joel Coehoorn' 
		and v.id > 0
	group by ps.id
) as cv;

-- 12
```


## Задание №5
Выгрузите все поля таблицы `vote_types`. 
Добавьте к таблице поле `rank`, в которое войдут номера записей в обратном порядке. 
Таблица должна быть отсортирована по полю `id`.

```sql
select id,
	name,
	row_number() over(order by id desc) as rank
from stackoverflow.vote_types
order by id;
```
```text
|id | name                  | rank |
|---|-----------------------|------|
| 1 | AcceptedByOriginator  | 15   |
| 2 | UpMod                 | 14   |
| 3 | DownMod               | 13   |
| 4 | Offensive             | 12   |
| 5 | Favorite              | 11   |
...
```


## Задание №6
Отберите 10 пользователей, которые поставили больше всего голосов типа `Close`. 
Отобразите таблицу из двух полей: идентификатором пользователя и количеством голосов. 
Отсортируйте данные сначала по убыванию количества голосов, потом по убыванию значения идентификатора пользователя.

```sql
select 
	u.id as id_vote,
    count(v.id) as vote_cnt
from stackoverflow.users u
join stackoverflow.votes v on u.id = v.user_id
join stackoverflow.vote_types vt on v.vote_type_id = vt.id
where vt.name = 'Close'
group by u.id
order by vote_cnt desc, id_vote desc
limit 10;
```
```text
| id_vote | vote_cnt |
|---------|----------|
| 20646   | 36       |
| 14728   | 36       |
| 27163   | 29       |
| 41158   | 24       |
| 24820   | 23       |
| 9345    | 23       |
| 3241    | 23       |
| 44330   | 20       |
| 38426   | 19       |
| 19074   | 19       |
```


## Задание №7
Отберите 10 пользователей по количеству значков, полученных в период с 15 ноября по 15 декабря 2008 года включительно.
Отобразите несколько полей:
- идентификатор пользователя;
- число значков;
- место в рейтинге — чем больше значков, тем выше рейтинг.
Пользователям, которые набрали одинаковое количество значков, присвойте одно и то же место в рейтинге.
Отсортируйте записи по количеству значков по убыванию, а затем по возрастанию значения идентификатора пользователя.

```sql
with users as (
    select b.user_id, count(b.id) as badges_cnt
    from stackoverflow.badges b
    where b.creation_date::date between '2008-11-15' and '2008-12-15'
    group by b.user_id
    order by badges_cnt desc
    limit 10
)
select 
    u.user_id,
    u.badges_cnt,
    dense_rank() over (order by u.badges_cnt desc) as rating
from users u
```
```text
| user_id | badges_cnt | rating |
|---------|------------|--------| 
| 22656   | 149        | 1      | 
| 34509   | 45         | 2      |
| 1288    | 40         | 3      |
| 5190    | 31         | 4      |
| 13913   | 30         | 5      |
| 893     | 28         | 6      |
| 10661   | 28         | 6      |
| 33213   | 25         | 7      |
| 12950   | 23         | 8      |
| 25222   | 20         | 9      |
```


## Задание №8
Сколько в среднем очков получает пост каждого пользователя?

Сформируйте таблицу из следующих полей:
- заголовок поста;
- идентификатор пользователя;
- число очков поста;
- среднее число очков пользователя за пост, округлённое до целого числа.

Не учитывайте посты без заголовка, а также те, что набрали ноль очков.

```sql
select p.title as title,
       p.user_id as user,
       p.score as score_posts,
       round(avg(p.score) over(partition by p.user_id)) as avg_score
from stackoverflow.users u
join stackoverflow.posts p on u.id = p.user_id
where p.title is not null and p.score != 0
```
```text
| title                                                               | user | score_posts | avg_score |
|---------------------------------------------------------------------|------|-------------|-----------| 
| Diagnosing Deadlocks in SQL Server 2005                             | 1    | 82          | 573       |
| How do I calculate someone's age in C#?                             | 1    | 1743        | 573       |
| Why doesn't IE7 copy <pre><code> blocks to the clipboard correctly? | 1    | 37          | 573       |
| Calculate relative time in C#                                       | 1    | 1348        | 573       |
| Wrapping StopWatch timing with a delegate or lambda?                | 1    | 92          | 573       |
| Practical non-image based CAPTCHA approaches?                       | 1    | 318         | 573       |
...
```


## Задание №9
Отобразите заголовки постов, которые были написаны пользователями, 
получившими более 1000 значков. Посты без заголовков не должны попасть в список.

```sql
with users_1000_badges as (
  select user_id
  from stackoverflow.badges
  group by user_id
  having count(id) > 1000
)   
select p.title
from stackoverflow.posts p
join users_1000_badges ub on p.user_id = ub.user_id
where p.title is not null
```
```text
| title                                                       |
|-------------------------------------------------------------|
| What's the strangest corner case you've seen in C# or .NET? |
| What's the hardest or most misunderstood aspect of LINQ?    |
| What are the correct version numbers for C#?                |
| Project management to go with GitHub                        |
```


## Задание №10
Напишите запрос, который выгрузит данные о пользователях из Канады (англ. Canada). 
Разделите пользователей на три группы в зависимости от количества просмотров их профилей:
- пользователям с числом просмотров больше либо равным 350 присвойте группу `1`;
- пользователям с числом просмотров меньше 350, но больше либо равно 100 — группу `2`;
- пользователям с числом просмотров меньше 100 — группу `3`.
Отобразите в итоговой таблице идентификатор пользователя, количество просмотров профиля и группу. 
Пользователи с количеством просмотров меньше либо равным нулю не должны войти в итоговую таблицу.

```sql
select 
  id,
  views,
  case
    when views >= 350 then 1
    when views < 350 and views >= 100 then 2
    when views < 100 then 3
  end as group
from stackoverflow.users
where 
  location like '%Canada%'
  and views > 0
```
```text
| id | views | group | 
|----|-------|-------|
| 22 | 1079  | 1     |
| 34 | 1707  | 1     |
| 37 | 757   | 1     |
| 41 | 174   | 2     |
...
```


## Задание №11
Дополните предыдущий запрос. 
Отобразите лидеров каждой группы — пользователей, которые набрали максимальное число просмотров в своей группе. 
Выведите поля с идентификатором пользователя, группой и количеством просмотров. 
Отсортируйте таблицу по убыванию просмотров, а затем по возрастанию значения идентификатора.

```sql
with gv as (
	select 
	  id,
	  views,
	  case
		when views >= 350 then 1
		when views < 350 and views >= 100 then 2
		when views < 100 then 3
	  end as "group"
	from stackoverflow.users
	where location like '%Canada%' and views > 0
),
grp as (
  select 
    id,
    "group",
    views,
    max(views) over(partition by "group") as views_max
  from gv
)
select id, views, "group"
from grp
where views = views_max
order by views desc, id;
```
```text
| id     | views | group |
|--------|-------|-------|
| 3153   | 21991 | 1     |
| 46981  | 349   | 2     |
| 3444   | 99    | 3     |
| 22273  | 99    | 3     |
| 190298 | 99    | 3     |
```


## Задание №12
Посчитайте ежедневный прирост новых пользователей в ноябре 2008 года. 
Сформируйте таблицу с полями:
- номер дня;
- число пользователей, зарегистрированных в этот день;
- сумму пользователей с накоплением.

```sql
with users_day as (
    select 
	  count(id) users_cnt, 
      cast(creation_date as date) as day_reg
    from stackoverflow.users
    where creation_date::date between '2008-11-01' and '2008-11-30'
    group by day_reg
    order by day_reg)
select
    row_number() over() as day_number,
    users_cnt,
    sum(users_cnt) over(order by day_reg) as users_sum
from users_day   
```
```text
| day_number | users_cnt | users_sum |
|------------|-----------|-----------|
| 1          | 34        | 34        |
| 2          | 48        | 82        |
| 3          | 75        | 157       |
| 4          | 192       | 349       |
```


## Задание №13
Для каждого пользователя, который написал хотя бы один пост, 
найдите интервал между регистрацией и временем создания первого поста. 

Отобразите:
- идентификатор пользователя;
- разницу во времени между регистрацией и первым постом.

```sql
with user_posts as (
    select user_id,
      min(creation_date) as first_post_dt
    from stackoverflow.posts
    group by user_id)
select 
  u.id,
  (up.first_post_dt - u.creation_date) as day_diff
from stackoverflow.users u
join user_posts up on u.id = up.user_id
```
```text
| id | day_diff         |
|----|------------------
| 1  | 9:18:29          |
| 2  | 14:37:03         |
| 3  | 3 days, 16:17:09 |
| 4  | 15 days, 5:44:22 |
...
```
