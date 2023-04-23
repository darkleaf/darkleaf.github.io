# Базы данных и приложения

+ [Postgresso 24](https://habr.com/ru/companies/postgrespro/articles/513632/)
+ [Как Lingualeo переехал на PostgreSQL с 23 млн юзеров](https://habr.com/ru/companies/lingualeo/articles/515530/)
+ [PgConf.Russia 2020](https://pgconf.ru/2020/264859)
+ [Postgres-вторник №10](https://www.youtube.com/watch?v=ZFLs7-vmbdM)


Какого вида хранимки? Я вот тоже не хочу на pl/pgSQL писать, а что если это просто LANGUAGE SQL?



CTE modifying
т.е. инсерты и аптдейты в cte.


Если 90% хранимок это sql, то зачем нужны хранимки?

А нужны они, чтобы запросы были подготовленными.
Если исползовать pg_pool и ко, то невозможно использовать prepared statement, и мы возвращаемся к хранимкам.


задачка

Есть посты, есть комментарии. нунжо вывести список постов с последним комментарием (id, текст).
Тут lateral join.

А теперь все тоже самое, но 2 последних комментария.
еще один lateral join, и 2 столбца?
массив id и массив текста?
json.

типизация в json. `["$kv" "ns/key"]`
https://github.com/metosin/jsonista#tagged-json

сделать хранимку `instant(x jsonb) -> timestamptz`,  `instant(x timestamptz) -> jsonb`.
и она делает `["$instant" 123456789]::jsonb`. Тоже самое с keyword, и прочими типами дат.


сделать insert из json `[{name: "xyz"}, {...}]`.

придумать как сделать вложенный инсерт `[{coll: [{...}, ...]}, ...]`. Видимо на CTE и join.

```sql
create table posts (
  id serial primary key,
  title varchar(256),
  content text
);

create table comments (
  id serial primary key,
  post_id integer,
  content text
);


insert into posts
select
  nextval('posts_id_seq') as id,
  title,
  content
from json_populate_recordset(
  null::posts,
  '[{"title":"a", "content":"xyz"},
    {"title":"b", "content":"zyx"}]'::json
)
returning id;

select * from posts;
select * from comments;
```

```sql
with data as (
  select * from json_array_elements(
 '[{
    "title":"a",
    "content":"xyz",
    "comments":[{"content":"123"}, {"content":"asd"}]
   },
   {
    "title":"b",
    "content":"zyx",
    "comments":[{"content":"987"}]
   }]'::json
  )
),
posts as (
  select
    nextval('posts_id_seq') as id,
    title,
    content,
    value->'comments' as comments
  from data, json_populate_record(null::posts, value)
),
insert_posts as (
  insert into posts
  select id, title, content
  from posts
  returning id
),
comments as (
  select
    nextval('comments_id_seq') as id,
    posts.id as posts_id,
    comments.content
  from posts,
       json_populate_recordset(null::comments, comments) as comments
),
insert_comments as (
  insert into comments
  select *
  from comments
  returning id
)
select 1;

select * from posts;
select * from comments;
```



Типы все равно нужно будет вручную приводить.
И придется 3 раза перечислять поля.

```sql
insert into posts (title, content, published_at)
select
  title,
  content,
  instant(published_at) as published_at
from json_to_record($1) as (title text, content text, published_at json)
```

`published_at = ["$instant", "2023-...."]::json`
