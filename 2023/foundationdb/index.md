# FoundationDB

Я пишу эту статью для себя в будущем. И пишу уже по воспоминаниям, спустя пару месяцев после знакомства с этой БД.

Когда я только начинал работать с Clickhouse, у меня постоянно возникала мысль:
"Дайте мне уже API к отсортированным данным и я сам все сделаю, я не хочу разбираться в приколах вашего SQL".
Со временем меня отпустило, т.к. стал лучше понимать задачи и их решения в Clickhouse.

Я периодически смотрю какие бывают базы данных, и однажды набрел на [DevZen: YDb, Санта и Кодзима](https://devzen.ru/episode-0272/),
где в том числе упоминалась FoundationDB. И я решил посмотреть на нее.

Так вот, FoundationDB как раз об этом. Там есть отсортированые ключи и ACID транзакции, а дальше сам.
Но все это сделано очень круто.

Все множество ключей само размазывается по кластеру, не нужно делать шардинг.
Если на какой-то ноде стало больше данных - часть данных переедет на другой сервер.

Другая не менее важная фишка - тестирование.
А именно возможность легко описывать надежно воспроизводимые сценарии сбоев в вводе-выводе.
БД написана на Flow, это такое надмножество C++.
Думаю, тут можно провести параллель с генераторами в JS и прочими континуациями и сопрограммами,
когда можно отделить работу с побочными эффектами и написать пошаговый тест.
Aphyr вообще отказался делать Japsen, сказал, что родные тесты на голову лучше его.

Изначально это была коммерческая БД. Затем в 2015 ее купила Apple.
И с 2018 она доступна под Apache License 2.0.

Я даже где-то видел комментарий, что FoundationDB это одно из современных чудес.

Название у БД не спроста такое. Идея в том, что есть надежное основание (foundation), поверх которого строятся слои (layers).
Слой это либо сетевой сервис, либо библиотека.
Из коробки есть утилиты, но можно назвать их слоями, для работы с таплами, подпространствами и директориями.
Есть слой для работы с записями на основе ProtoBuf: [record-layer](https://github.com/FoundationDB/fdb-record-layer).
Были еще документные и графовые слои, но они перестали развиваться.

[Документация](https://apple.github.io/foundationdb/) почти идеальная. Она глубокая и по делу, было довольно интересно и познавательно ее читать.
Кроме документации есть [формум](https://forums.foundationdb.org/), где разработчики очень подробно разбирают вопросы.
Подробно, это вот так: [What’s the purpose of the Directory layer?](https://forums.foundationdb.org/t/whats-the-purpose-of-the-directory-layer/677).
Еще есть видео с пары конференций, хотя они и старые по современным меркам, но актуальность не потеряли.

Для того, чтобы все это работало, нужен довольно сложный клиент. Как минимум клиент следит за нодами и сам повторяет транзакции.
Так вот, клиент написан на C и используеется из всех других библиотек, включая Java.

Помимо базовых операций вроде get/put есть набор мутаций, например, можно инкрементировать счетчик, не вызывая конфликтов.
Еще есть [мутации](https://apple.github.io/foundationdb/javadoc/com/apple/foundationdb/MutationType.html),
например, можно изменить счетчик на заданную дельту, сгенерировать уникальный идентификатор и пололжить его в ключ или значение: `SET_VERSIONSTAMPED_KEY`/`SET_VERSIONSTAMPED_VALUE`.

Versionstamp - это уникальный в рамках кластера идентификатор транзакции размером в 10 байт.
Еще добавляют 2 байта для счетчика в приложении.
Т.е. тут нет проблем с генерацией монотонных идентификаторов, и он относительно короткий, всего 10+2 байта вместо 16 у UUID.

Тут [нет хранимых процедур](https://forums.foundationdb.org/t/stored-procedures/1993).
И для сложных операций придется гонять туда-сюда много данных.
Когда я выше присал про почти идельную документацию, я написал почти, т.к. не понял из документации как работать с непокрывающими индексами.
Индексы бывают покрывающие и непокрывающие. Если для выполнения запроса достаточно прочитать только индекс, то такой индекс называется покрывающим для этого запроса.
Непокрывающие индексы, наверное можно назвать косвенными, т.к. требуют дополнительного **случайного** чтения, и это проблема N+1.
Но разработчики знают о этой проблеме и добавили эксперементальный метод [`getMappedRange`](https://github.com/apple/foundationdb/wiki/Everything-about-GetMappedRange), который делает эти косвенные запросы.

В моем представлении FoundationDB отлично подойдет нагруженным проетам с простой предметной областью.
Различные SaaS вроде таск-трекеров, документооборота, социальных сетей.
Когда очень много пользователей, много данных, и нужны транзакции между разными "шардами".
Т.е. не нужно думать, как шардировать данные, как их решадировать, когда клиент вырос из шарда,
и что самое важное, не нужно думать, как организовать транзакции между шардами.
Решения вроде паттерна [Saga](https://microservices.io/patterns/data/saga.html) становятся не нужны или отодвигаются в будущее.

## Таск-трекер

Например, есть таск-трекер по типу канбан доски.
Там есть мини карточка и собственно сама карточка с описанием, комментариеями и т.п.

Ниже я буду использовать нотацию `{some_id}:some_label -> some data` чтобы показать структуру ключей и значений.
В примере `{some_id}` это подстановка некоторого id, а `some_label` некий литерал.

Так вот мини карточку можно хранить денормализовано и сделать по ней покрывающий индекс:
+ `{project_id}:mini_card_by_tag:{tag_id}:{card_id} -> mini_card data`
  так мы одним запросом по диапазону найдем карточки с тегом, `card_id` нужен, чтобы для одного тега хранилось несколько кароточек
+ `{project_id}:mini_card_by_author:{author_id}:{card_id} -> mini_card data`
   аналогично по автору
+ и так далее для всех интересующих полей

А полную карточку уже можно хранить только один раз и обращаться к ней по `card_id`, ведь это исходит из концепции интерфейса пользователя.

## Резюме

FoundationDB используется Apple для сервисов iCloud. Представьте масштаб.

Все это очень интересно, но большинству проектов это не нужно.
Я был бы очень рад, если бы нагрузку моего проекта не смог обслужить Postgres с сотней ядер, полтерабата RAM и бырыми локальными SSD.
С другой стороны, если будет нужно делать новый Twitter или Asana, я знаю первого кандидата на роль БД.

Вот еще кстати хорошее видео от [Максима Нальского (Pyrus)](https://www.youtube.com/watch?v=zv8LGlQ6xco).

# Datomic

Опциональная часть.

Еще я думал, что было бы интересно реализовать слой, похожий на [Datomic](https://docs.datomic.com/cloud/whatis/data-model.html).
Это очень интересная коммерческая база данных от автора Clojure - Rich Hickey.

Данные хранятся фактами в виде троек: идентификатора сущности, атрибута и значения. На самом деле там еще есть номер транзакции и признак удаления, но это не важно.

| E  | A                    | V          |
|----|----------------------|------------|
| 42 | :user/favorite-color | :blue      |
| 42 | :user/first-name     | "John"     |
| 42 | :user/last-name      | "Doe"      |
| 42 | :user/favorite-color | :green	 |
| 42 | :user/favorite-color | :blue      |

И все данные хранятся в [4х покрывающих индексах](https://docs.datomic.com/cloud/query/raw-index-access.html#indexes).
Используется сканирование по отсортированным тройкам.

+ EAV, аналог строковых БД, для поиска всех атрибутов сущности по ее id
+ AEV, аналог колончатых БД, для поиска всех значений колоки/атрибута
+ AVE, для индексируемых атрибутов
+ VAE, для поиска по ссылкам

Заметка по хранению из черновика:

+ eav можно хранить как ea->v для one и eav-> для many
+ aev можно хранить как ea->v для one и eav-> для many
+ ave только как ave->
+ vae только как vae-> тут всегда 12+2+12 = 26байт или 12 + 12 + 12 = 36

Я уже точно не помню, но проблема была с [composite tuples](https://docs.datomic.com/cloud/schema/schema-reference.html#composite-tuples).
В Datomic можно объявить объявить атрибут, который будет содержать тапл из других атрибутов этой сущсности.
И если мы записываем только один датом, а не всю сущность, то нужны дополнительне телодвижения для обновления
композитного тапла в 3х индексах.

В итоге я отказался от модели Datomic, т.к. это все-таки общее решение и нет особого смысла пытаться его повторить, когда можно сделать простое частное решение для конкретной задачи.
Но как мысленный эксперимент вполне ок.