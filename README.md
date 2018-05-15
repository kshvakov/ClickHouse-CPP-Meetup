ClickHouse C++ Meetup (2018-05-16)
===

# ClickHouse tips & tricks

Материалы к докладу на ClickHouse C++ Meetup в Яндекс.

Всё сказанное, по большей части, справедливо для ClickHouse `1.1.54343` и, если не сказано обратного, используется на практике в [TrafficStars](https://trafficstars.com/).

* [О проекте](#trafficstars)
* [ClickHouse table engines](#clickHouse-table-engines-)
    1) [CollapsingMergeTree](#collapsingmergetree)
    2) [SummingMergeTree](#summingmergetree)
    3) [MergeTree](#mergetree)
    4) [Транзакции](#Транзакции)
* [Материализованные представления](#Материализованные-представления)
* [Миграции](#Миграции) 
* [Мониторинг](#Мониторинг)

## TrafficStars

Рекламная сетка, на данный момент в ClickHouse пишется порядка 1.5 миллиардов событий в сутки, при этом пиковая нагрузка до 30K rps. 

Datawlow (упрощённо):

* все события пишем в Kafka 
* закрепленный за шардом воркер читает события из Kafka, производит обработку события, собирает их в "пачку" и пишет в ClickHouse 
* Раз в N запускаются различные процессы для построения обновления пользовательских отчетов


1) мы не пишем в distributed-таблицы
2) пользователи не работают с сырыми данными

## ClickHouse table engines :)

### CollapsingMergeTree

Все сырые события пишутся в таблицу с движком `CollapsingMergeTree`. Для нас важно что данный движок во время мержей удаляет дубликаты по первичному ключу, а так же то, что он позволяет удалять данные (никто не застрахован от ошибок). 

**Удаление дубликатов.**

В процессе работы у нас неизбежно появляются дубликаты, с первичной очисткой данных прекрасно справляется `CollapsingMergeTree`. 

Пример удаления дубликатов. 

```sql
/* Создаем таблицу сырых событий*/
CREATE TABLE RawEvents (
      EventID   FixedString(16)
    , EventTime DateTime
    , EventDate MATERIALIZED toDate(EventTime)
    , Price     Int64 
    , Sign      Int8 DEFAULT 1
) Engine = CollapsingMergeTree(Sign)
PARTITION BY toYYYYMM(EventDate)
ORDER BY (
    EventDate,
    EventID
);

/* Вставляем данные с дубликатами */
INSERT INTO RawEvents (EventID, EventTime, Price) 
    VALUES 
          ('XXXXXXXXXXXXXXXX', '2018-05-09 15:00:00', 100)
        , ('YYYYYYYYYYYYYYYY', '2018-05-09 15:00:00', 100)
        , ('YYYYYYYYYYYYYYYY', '2018-05-09 15:00:00', 100);

INSERT INTO RawEvents (EventID, EventTime, Price) 
    VALUES 
          ('ZZZZZZZZZZZZZZZZ', '2018-05-09 15:00:00', 100)
        , ('XXXXXXXXXXXXXXXX', '2018-05-09 15:00:00', 100)
        , ('AAAAAAAAAAAAAAAA', '2018-05-09 15:00:00', 100); 

/* 
    При мержах CollapsingMergeTree удалит дубликаты записей по первичному ключу.
    В нашем случае это -  ('2018-05-09', 'XXXXXXXXXXXXXXXX').
    Чтоб не дожидаться мержа выполняем запрос с FINAL
*/
localhost :) SELECT * FROM RawEvents FINAL ORDER BY EventID 

┌─EventID──────────┬───────────EventTime─┬─Price─┬─Sign─┐
│ AAAAAAAAAAAAAAAA │ 2018-05-09 15:00:00 │   100 │    1 │
│ XXXXXXXXXXXXXXXX │ 2018-05-09 15:00:00 │   100 │    1 │
│ YYYYYYYYYYYYYYYY │ 2018-05-09 15:00:00 │   100 │    1 │
│ ZZZZZZZZZZZZZZZZ │ 2018-05-09 15:00:00 │   100 │    1 │
└──────────────────┴─────────────────────┴───────┴──────┘

4 rows in set. Elapsed: 0.009 sec.
```

**Удаление строк.**

Помимо дубликатов всегда существует опастность того, что сырые данные могут оказаться неверными, например в случае какой-нибудь ошибки в нашем ПО. Хранить заведомо некоректные данные нет никакого смысла, поэтому мы достаточно легко можем их удалить.

```sql

/* Удаляем строку с первичным ключём ('2018-05-09', 'YYYYYYYYYYYYYYYY')*/

INSERT INTO RawEvents (EventID, EventTime, Price, Sign) 
    VALUES 
        ('YYYYYYYYYYYYYYYY', '2018-05-09 15:00:00', 100, -1); 

┌─EventID──────────┬───────────EventTime─┬─Price─┬─Sign─┐
│ AAAAAAAAAAAAAAAA │ 2018-05-09 15:00:00 │   100 │    1 │
│ XXXXXXXXXXXXXXXX │ 2018-05-09 15:00:00 │   100 │    1 │
│ ZZZZZZZZZZZZZZZZ │ 2018-05-09 15:00:00 │   100 │    1 │
└──────────────────┴─────────────────────┴───────┴──────┘

3 rows in set. Elapsed: 0.005 sec.
```

### SummingMergeTree

Пожалуй, самый важный из движков для нас. Несмотря на то, что, в большинстве случаев, ClickHouse позволяет строить запросы к неагрегированным/сырам данным мы этого не делаем. 
Во-первых, мы не можем отдавать пользовательскую статистику по сырым данным, так как нам необходима некоторая постобработка. 
Во-вторых, у нас достаточно много запросов на чтение и не так много железа чтоб справиться с их обработкой. 

Изначально у нас были отдельные таблицы под разные отчеты + мы отдельно хранили почасовую и посуточную статистику. На данный момент для хранения стандартных отчетов мы используем `Nested` структуры и храним всё в одной таблице.

Особенностью `SummingMergeTree` является то, что он суммирует значения в колонках с группировкой по первичному ключу, в том числе и в `Nested` структурах.

Эта особенность является важнейшей при выборе этого движка для хранения отчетов так как позволяет нам агрегировать данные в фоне которые, в настоящий момент, мы не можем агрегировать самостоятельно (нам попросту не хватает памяти на агрегацию массивов, например при использовании `sumMap`). 

```sql
/* Создаем таблицу для сырых событий */

DROP TABLE IF EXISTS RawEvents;

CREATE TABLE RawEvents (
      EventTime DateTime
    , EventDate Date MATERIALIZED toDate(EventTime)
    , Price     Int64
    , ClientID  Int64
    , BrowserID Int64
    , CountryID Int64
) Engine = MergeTree 
PARTITION BY tuple()
ORDER     BY (EventDate);

/* Запишем события */

INSERT INTO RawEvents (
    EventTime
    , Price
    , ClientID
    , BrowserID
    , CountryID
) VALUES 
      ('2018-04-09 14:00:00', 12, 1, 1, 8)
    , ('2018-04-09 14:00:00', 25, 1, 2, 1)
    , ('2018-04-09 14:00:00', 11, 1, 5, 4)
    , ('2018-04-09 14:00:00', 44, 1, 3, 5)
    , ('2018-04-09 15:00:00', 16, 1, 1, 1)
    , ('2018-04-09 15:00:00', 88, 1, 2, 1)
    , ('2018-04-09 15:00:00', 33, 1, 2, 2)
    , ('2018-04-09 15:00:00', 42, 1, 3, 5)
    , ('2018-04-09 14:00:00', 56, 2, 11, 8)
    , ('2018-04-09 14:00:00', 54, 2, 2, 1)
    , ('2018-04-09 14:00:00', 9,  2, 5, 14)
    , ('2018-04-09 14:00:00', 11, 2, 3, 5)
    , ('2018-04-09 15:00:00', 22, 2, 1, 3)
    , ('2018-04-09 15:00:00', 36, 2, 8, 1)
    , ('2018-04-09 15:00:00', 54, 2, 2, 2)
    , ('2018-04-09 15:00:00', 87, 2, 3, 5)
    , ('2018-05-09 14:00:00', 12, 3, 1, 8)
    , ('2018-05-09 14:00:00', 11, 3, 2, 1)
    , ('2018-05-09 14:00:00', 71, 7, 5, 4)
    , ('2018-05-09 14:00:00', 12, 1, 3, 5)
    , ('2018-05-09 15:00:00', 58, 1, 1, 1)
    , ('2018-05-09 15:00:00', 32, 1, 2, 1)
    , ('2018-05-09 15:00:00', 11, 1, 2, 2)
    , ('2018-05-09 15:00:00', 10, 1, 3, 5)
    , ('2018-05-09 14:00:00', 12, 2, 11, 8)
    , ('2018-05-09 14:00:00', 22, 2, 2, 1)
    , ('2018-05-09 14:00:00', 8,  2, 5, 14)
    , ('2018-05-09 14:00:00', 98, 2, 3, 5)
    , ('2018-05-09 15:00:00', 66, 2, 1, 3)
    , ('2018-05-09 15:00:00', 33, 2, 8, 1)
    , ('2018-05-09 15:00:00', 17, 2, 2, 2)
    , ('2018-05-09 15:00:00', 88, 2, 3, 5);
```

Как уже говорилось выше - все простые отчёты мы храним в одной таблице в `Nested` структурах.
Для примера создадим таблицу, которая содержит общий агрегированный отчет, а также отчёты по браузеру и стране.

 ```sql
CREATE TABLE Reports (
      EventTime   DateTime DEFAULT toDateTime(EventDate)
    , EventDate   Date
    , ClientID    Int64
    , Price       Int64
    , Impressions Int64
    , BrowserMap Nested (
          ID          Int64
        , Price       Int64
        , Impressions Int64
    )
    , CountryMap Nested (
          ID          Int64
        , Price       Int64
        , Impressions Int64
    )
    , FakeRow Int8
) Engine = SummingMergeTree (
    (
        Price, Impressions
    )
)
PARTITION BY toYYYYMM(EventDate)
ORDER BY (
    ClientID,
    EventDate,
    EventTime
);
 ```

Далее мы можем просто писать данные без предварительной агрегации.

```sql
INSERT INTO Reports (
      EventDate
    , EventTime
    , ClientID
    , Price
    , Impressions
    , "BrowserMap.ID"
    , "BrowserMap.Price"
    , "BrowserMap.Impressions"
    , "CountryMap.ID"
    , "CountryMap.Price"
    , "CountryMap.Impressions"
)
SELECT 
       toDate(EventTime)
    ,  toStartOfHour(EventTime) AS EventTime
    , ClientID
    , Price
    , CAST(1 AS Int64)   AS Impressions /* каждая запись - это событие, нам нужно посчитать общее количество */
    , [BrowserID]        AS "BrowserMap.ID"
    , [Price]            AS "BrowserMap.Price"
    , [Impressions]      AS "BrowserMap.Impressions"
    , [CountryID]        AS "CountryMap.ID"
    , [Price]            AS "CountryMap.Price"
    , [Impressions]      AS "CountryMap.Impressions"
FROM RawEvents;
```

Мы не будем ждать пока `ClickHouse` выполнит слияние в фоне и немного ему поможем.

```sql
/*
Чтоб сработал мерж запишем еще блок данных
*/
INSERT INTO Reports (EventDate, FakeRow) VALUES ('2018-04-09', 1);
INSERT INTO Reports (EventDate, FakeRow) VALUES ('2018-05-09', 1);

OPTIMIZE TABLE Reports PARTITION 201804 FINAL;
OPTIMIZE TABLE Reports PARTITION 201805 FINAL;
```

ОК. Проверим что все данные проссумировались и они коректные

```sql
localhost :) SELECT EventDate, EventTime, ClientID, Impressions, Price FROM Reports WHERE FakeRow = 0 ORDER BY EventTime

┌──EventDate─┬───────────EventTime─┬─ClientID─┬─Impressions─┬─Price─┐
│ 2018-04-09 │ 2018-04-09 14:00:00 │        1 │           4 │    92 │
│ 2018-04-09 │ 2018-04-09 14:00:00 │        2 │           4 │   130 │
│ 2018-04-09 │ 2018-04-09 15:00:00 │        1 │           4 │   179 │
│ 2018-04-09 │ 2018-04-09 15:00:00 │        2 │           4 │   199 │
│ 2018-05-09 │ 2018-05-09 14:00:00 │        1 │           1 │    12 │
│ 2018-05-09 │ 2018-05-09 14:00:00 │        2 │           4 │   140 │
│ 2018-05-09 │ 2018-05-09 14:00:00 │        3 │           2 │    23 │
│ 2018-05-09 │ 2018-05-09 14:00:00 │        7 │           1 │    71 │
│ 2018-05-09 │ 2018-05-09 15:00:00 │        1 │           4 │   111 │
│ 2018-05-09 │ 2018-05-09 15:00:00 │        2 │           4 │   204 │
└────────────┴─────────────────────┴──────────┴─────────────┴───────┘
```

Теперь проверим, на примере клиента #2 всё ли посчиталось правильно, и построим отчет по браузерам.

```sql
localhost :) SELECT EventTime, ClientID, SUM(Impressions) AS Imps, SUM(Price) AS Price FROM Reports WHERE ClientID = 2 GROUP BY EventTime, ClientID

┌───────────EventTime─┬─ClientID─┬─Imps─┬─Price─┐
│ 2018-05-09 15:00:00 │        2 │    4 │   204 │
│ 2018-05-09 14:00:00 │        2 │    4 │   140 │
│ 2018-04-09 15:00:00 │        2 │    4 │   199 │
│ 2018-04-09 14:00:00 │        2 │    4 │   130 │
└─────────────────────┴──────────┴──────┴───────┘

/* Посмотрим, совпадают ли общие данные и данные в отчете по браузерам */

localhost :)  
SELECT 
    EventTime, 
    ClientID, 
    SUM(BrowserMap.Impressions) AS Imps, 
    SUM(BrowserMap.Price) AS Price
FROM Reports 
ARRAY JOIN BrowserMap
WHERE ClientID = 2
GROUP BY 
    EventTime, 
    ClientID

┌───────────EventTime─┬─ClientID─┬─Imps─┬─Price─┐
│ 2018-05-09 15:00:00 │        2 │    4 │   204 │
│ 2018-05-09 14:00:00 │        2 │    4 │   140 │
│ 2018-04-09 15:00:00 │        2 │    4 │   199 │
│ 2018-04-09 14:00:00 │        2 │    4 │   130 │
└─────────────────────┴──────────┴──────┴───────┘
```

Похоже что всё хорошо, данные совпадают, теперь самое время построить отчет с разбивкой по часам и браузерам :)

```sql
localhost :)  
SELECT 
    EventTime, 
    ClientID, 
    BrowserMap.ID AS BrowserID, 
    SUM(BrowserMap.Impressions) AS Imps, 
    SUM(BrowserMap.Price) AS Price
FROM Reports 
ARRAY JOIN BrowserMap
WHERE ClientID = 2
GROUP BY 
    EventTime, 
    ClientID, 
    BrowserID
    WITH TOTALS
ORDER BY 
    EventTime ASC, 
    BrowserID ASC
┌───────────EventTime─┬─ClientID─┬─BrowserID─┬─Imps─┬─Price─┐
│ 2018-04-09 14:00:00 │        2 │         2 │    1 │    54 │
│ 2018-04-09 14:00:00 │        2 │         3 │    1 │    11 │
│ 2018-04-09 14:00:00 │        2 │         5 │    1 │     9 │
│ 2018-04-09 14:00:00 │        2 │        11 │    1 │    56 │
│ 2018-04-09 15:00:00 │        2 │         1 │    1 │    22 │
│ 2018-04-09 15:00:00 │        2 │         2 │    1 │    54 │
│ 2018-04-09 15:00:00 │        2 │         3 │    1 │    87 │
│ 2018-04-09 15:00:00 │        2 │         8 │    1 │    36 │
│ 2018-05-09 14:00:00 │        2 │         2 │    1 │    22 │
│ 2018-05-09 14:00:00 │        2 │         3 │    1 │    98 │
│ 2018-05-09 14:00:00 │        2 │         5 │    1 │     8 │
│ 2018-05-09 14:00:00 │        2 │        11 │    1 │    12 │
│ 2018-05-09 15:00:00 │        2 │         1 │    1 │    66 │
│ 2018-05-09 15:00:00 │        2 │         2 │    1 │    17 │
│ 2018-05-09 15:00:00 │        2 │         3 │    1 │    88 │
│ 2018-05-09 15:00:00 │        2 │         8 │    1 │    33 │
└─────────────────────┴──────────┴───────────┴──────┴───────┘

Totals:
┌───────────EventTime─┬─ClientID─┬─BrowserID─┬─Imps─┬─Price─┐
│ 0000-00-00 00:00:00 │        0 │         0 │   16 │   673 │
└─────────────────────┴──────────┴───────────┴──────┴───────┘

16 rows in set. Elapsed: 0.009 sec.
```

**Удаление строк.**

Представим, что случилось так, что для клиента #2 данные за `2018-04-09` неправильны. Нам их нужно удалить. Тут всё достаточно просто, мы пишем данные так, чтоб в сумме они всегда давали 0 при агрегации.

```sql
INSERT INTO Reports (
      EventDate
    , EventTime
    , ClientID
    , Price
    , Impressions
    , "BrowserMap.ID"
    , "BrowserMap.Price"
    , "BrowserMap.Impressions"
    , "CountryMap.ID"
    , "CountryMap.Price"
    , "CountryMap.Impressions"
)
SELECT 
      EventDate
    , EventTime
    , ClientID
    , -1 * Price
    , -1 * Impressions
    , BrowserMap.ID
    , arrayMap(x -> (-1 * x), BrowserMap.Price)
    , arrayMap(x -> (-1 * x), BrowserMap.Impressions)
    , CountryMap.ID
    , arrayMap(x -> (-1 * x), CountryMap.Price)
    , arrayMap(x -> (-1 * x), CountryMap.Impressions)
FROM Reports
WHERE ClientID = 2 AND EventDate = '2018-04-09';

OPTIMIZE TABLE Reports PARTITION 201804 FINAL
```

На самом деле пример с `Nested` будет не самым удачным, т.к. после мерджа он не удалит строки совсем.  

```sql
localhost :) SELECT  * FROM Reports WHERE ClientID = 2 ORDER BY EventTime 

┌───────────EventTime─┬─ClientID─┬─Price─┬─Impressions─┬─BrowserMap.ID─┬─BrowserMap.Price─┬─BrowserMap.Impressions─┬─CountryMap.ID─┬─CountryMap.Price─┬─CountryMap.Impressions─┬─FakeRow─┐
│ 2018-04-09 14:00:00 │        2 │     0 │           0 │ []            │ []               │ []                     │ []            │ []               │ []                     │       0 │
│ 2018-04-09 15:00:00 │        2 │     0 │           0 │ []            │ []               │ []                     │ []            │ []               │ []                     │       0 │
│ 2018-05-09 14:00:00 │        2 │   140 │           4 │ [2,3,5,11]    │ [22,98,8,12]     │ [1,1,1,1]              │ [1,5,8,14]    │ [22,98,12,8]     │ [1,1,1,1]              │       0 │
│ 2018-05-09 15:00:00 │        2 │   204 │           4 │ [1,2,3,8]     │ [66,17,88,33]    │ [1,1,1,1]              │ [1,2,3,5]     │ [33,17,66,88]    │ [1,1,1,1]              │       0 │
└─────────────────────┴──────────┴───────┴─────────────┴───────────────┴──────────────────┴────────────────────────┴───────────────┴──────────────────┴────────────────────────┴─────────┘

4 rows in set. Elapsed: 0.009 sec. 
```

Тем не менее, данные в таблице очищены и наши отчеты корректны.

```sql
localhost :) 
SELECT 
    EventTime, 
    ClientID, 
    SUM(Impressions) AS Imps, 
    SUM(Price) AS Price
FROM Reports 
WHERE ClientID = 2
GROUP BY 
    EventTime, 
    ClientID
    WITH TOTALS
ORDER BY EventTime ASC

┌───────────EventTime─┬─ClientID─┬─Imps─┬─Price─┐
│ 2018-04-09 14:00:00 │        2 │    0 │     0 │
│ 2018-04-09 15:00:00 │        2 │    0 │     0 │
│ 2018-05-09 14:00:00 │        2 │    4 │   140 │
│ 2018-05-09 15:00:00 │        2 │    4 │   204 │
└─────────────────────┴──────────┴──────┴───────┘

Totals:
┌───────────EventTime─┬─ClientID─┬─Imps─┬─Price─┐
│ 0000-00-00 00:00:00 │        0 │    8 │   344 │
└─────────────────────┴──────────┴──────┴───────┘

4 rows in set. Elapsed: 0.004 sec.
```

Почасовые отчеты это хорошо, но, через какое-то время такая точность становиться не нужна и мы можем "схлопнуть" их до суток.

* Небольшая хитрость, при создании таблицы отчётов мы указали что 
```sql
EventTime DateTime DEFAULT toDateTime(EventDate)
```

```sql
localhost :) SELECT EventTime, ClientID, Impressions FROM Reports WHERE ClientID = 1

┌───────────EventTime─┬─ClientID─┬─Impressions─┐
│ 2018-04-09 14:00:00 │        1 │           4 │
│ 2018-04-09 15:00:00 │        1 │           4 │
└─────────────────────┴──────────┴─────────────┘
┌───────────EventTime─┬─ClientID─┬─Impressions─┐
│ 2018-05-09 14:00:00 │        1 │           1 │
│ 2018-05-09 15:00:00 │        1 │           4 │
└─────────────────────┴──────────┴─────────────┘

4 rows in set. Elapsed: 0.005 sec. 

```

Попробуем ещё немного "схлопнуть" данные при помощи ClickHouse, например, за апрель 2018.

```sql
ALTER TABLE Reports CLEAR COLUMN EventTime IN PARTITION 201804

/* Посмотрим что получилось */

localhost :) SELECT EventTime, ClientID, Impressions FROM Reports WHERE ClientID = 1 ORDER BY EventTime

┌───────────EventTime─┬─ClientID─┬─Impressions─┐
│ 2018-04-09 00:00:00 │        1 │           4 │
│ 2018-04-09 00:00:00 │        1 │           4 │
│ 2018-05-09 14:00:00 │        1 │           1 │
│ 2018-05-09 15:00:00 │        1 │           4 │
└─────────────────────┴──────────┴─────────────┘

4 rows in set. Elapsed: 0.004 sec. 
```

Это не совсем то что хотелось бы, но видим что `EventTime` стал такой как нам нужно, воспользуемся нашим "трюком" и принудительно запустим мерж 

```sql
INSERT INTO Reports (EventDate, FakeRow) VALUES ('2018-04-09', 1);

OPTIMIZE TABLE Reports PARTITION 201804 FINAL;
```

Проверим

```sql
localhost :)  SELECT EventTime, ClientID, Impressions FROM Reports WHERE ClientID = 1 ORDER BY EventTime

┌───────────EventTime─┬─ClientID─┬─Impressions─┐
│ 2018-04-09 00:00:00 │        1 │           8 │
│ 2018-05-09 14:00:00 │        1 │           1 │
│ 2018-05-09 15:00:00 │        1 │           4 │
└─────────────────────┴──────────┴─────────────┘
```

С `ReplicatedSummingMergeTree` данный трюк, пока, не пройдет. После `CLEAR COLUMN` мерж происходить не будет, но можно сделать `DETACH/ATTACH` партиции и всё будет ОК :)

### MergeTree

Используется для хранения уже обработанных и частично агрегированных данных полученных из сырых событий. 

Если нам необходимо изменить или удалить данные мы, как и в случае с `SummingMergeTree` просто пишем отрицательные значения.


### Транзакции

Их нет. 

Как уже говорилось выше - наши клиенты получают статистику не с сырых данных, отчеты предварительно генерируются и тут есть несколько моментов:

1) отчеты строятся достаточно долго 
2) таблиц отчетов несколько 

Если просто писать данные в таблицы:

* несогласованные данные в течении долгого времени
* существует вероятность того, что репорты будут посчитаны неправильно и данные разойдутся

Мы создаем копии таблиц и сохраняем отчет в них, после проверки что всё правильно мы перемещаем партиции (`rsync -ar`) в целевые таблицы и делаем `ATTACH`.

Это, конечно, лучше чем просто записывать данные в таблицы отчетов, но всё равно остаются проблемы:
* нам необходим доступ к файловой системе для перемещения партиции
* остается вероятность что на одной из таблиц `ATTACH` может не пройти

*для решения первой проблемы есть PR [Add ALTER TABLE REPLACE PARTITION FROM](https://github.com/yandex/ClickHouse/pull/2217)

### Материализованные представления

После появления `CREATE MATERIALIZED VIEW mv TO t` используем только их.

Как работают материализованные представления: 

* при создании MV выбираем к какой базе данных и таблице оно относится и привязываем к ней
* если не указано `TO` - создаем таблицу `.inner.` + название представления
* при вставке в таблицу смотрим какие представления к ней относятся и в цикле применяем запрос к блоку данных

При удалении представления проверяем создавалаять она с `.inner.`* и если да то удаляем и `.inner.` таблицу. 

*сейчас на эту проверку также завязаны запросы на работу с партициями 

Поэтому раньше было очень сложно обслуживать материализованные представления, для обновления требовалось изменять код представляения в `/var/lib/clickhouse/metadata/` и перезагружать сервер, либо копировать данные.

### Миграции & обслуживание 

* Не используем никакие известные и не известные системы миграций
* Не используем `ON CLUSTER`
* На время любых изменений в схеме хранения мы полностью отключаем запись в шард
* Удаление партиций/колонок с условием WHERE toDayOfWeek(today()) BETWEEN 1 AND 4

Реальный скрипт который заливает схемы данных на сервера :) 

 ```sh
#! /bin/bash 

if [ -z "$1" ]
  then
    echo "
Usage:
    ./load.sh server_address 
    "
    exit 0
fi

addr="$1"

dirs=(
    "tasks"
    "system"
    "actions"
    "reports"
    "views"
    "dictionaries"
)

clickhouse-client -n -h $addr < databases.sql

for dir in ${dirs[@]} 
do 
    for file in `ls ${dir} | grep \.sql`
    do
        script=$dir/$file
        if [ ! -d $script ]; then
            echo $script
            clickhouse-client -n -h $addr < $script
        fi
    done
done 
```


### Мониторинг

В ClickHouse большое внимание уделено метрикам работы сервера. Большую часть информации можно найти в системной схеме (базе данных):

:) SHOW TABLES FROM system

 * asynchronous_metrics
 * system.events 
 * merges
 * parts
 * query_log
 * replication_queue


У нас есть пишущие воркеры и API через которое идут запросы на чтение в ClickHouse от наших сервисов. 

Сейчас мы разделяем метрики для этих двух сервисов. 

Память и CPU понятное дело мониторить нужно, но, важным для нас является:
1) Для пишущего 
    * количество кусков в партиции `system.asynchronous_metrics MaxPartCountForPartition` и system.parts
    * репликация `system.asynchronous_metrics Replicas*`, `system.replication_queue`
2) Для читающего 
    * Попадание в кэш блоков system.events MarkCacheMisses/MarkCacheHits
    * Время выполнения и сами запросы `system.query_log`

Мы не получаем метрики только от `ClickHouse`, важным является то, что наши приложения так же отправляют метрики и мы их анализируем совместно с метриками от `ClickHouse`.

Мы используем следующие OpenSource компоненты: 
* [Grafana  ClickHouse datasource](https://github.com/Vertamedia/clickhouse-grafana)

* [ClickHouse Queries dashboard](https://grafana.com/dashboards/2515)

* [Clickhouse Exporter for Prometheus](https://github.com/f1yegor/clickhouse_exporter)

* Есть PR для Telegraf ([Add ClickHouse input plugin](https://github.com/influxdata/telegraf/pull/3894))
