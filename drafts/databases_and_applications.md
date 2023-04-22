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
