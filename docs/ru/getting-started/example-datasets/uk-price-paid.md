---
sidebar_position: 20
sidebar_label: Набор данных о стоимости недвижимости в Великобритании
---

# Набор данных о стоимости недвижимости в Великобритании {#uk-property-price-paid}

Набор содержит данные о стоимости недвижимости в Англии и Уэльсе. Данные доступны с 1995 года.
Размер набора данных в несжатом виде составляет около 4 GiB, а в ClickHouse он займет около 278 MiB.

Источник: https://www.gov.uk/government/statistical-data-sets/price-paid-data-downloads
Описание полей таблицы: https://www.gov.uk/guidance/about-the-price-paid-data

Набор содержит данные HM Land Registry data © Crown copyright and database right 2021. Эти данные лицензированы в соответствии с Open Government Licence v3.0.

## Загрузите набор данных {#download-dataset}

Выполните команду:

```bash
wget http://prod.publicdata.landregistry.gov.uk.s3-website-eu-west-1.amazonaws.com/pp-complete.csv
```

Загрузка займет около 2 минут при хорошем подключении к интернету.

## Создайте таблицу {#create-table}

```sql
CREATE TABLE uk_price_paid
(
    price UInt32,
    date Date,
    postcode1 LowCardinality(String),
    postcode2 LowCardinality(String),
    type Enum8('terraced' = 1, 'semi-detached' = 2, 'detached' = 3, 'flat' = 4, 'other' = 0),
    is_new UInt8,
    duration Enum8('freehold' = 1, 'leasehold' = 2, 'unknown' = 0),
    addr1 String,
    addr2 String,
    street LowCardinality(String),
    locality LowCardinality(String),
    town LowCardinality(String),
    district LowCardinality(String),
    county LowCardinality(String),
    category UInt8
) ENGINE = MergeTree ORDER BY (postcode1, postcode2, addr1, addr2);
```

## Обработайте и импортируйте данные {#preprocess-import-data}

В этом примере используется `clickhouse-local` для предварительной обработки данных и `clickhouse-client` для импорта данных.

Указывается структура исходных данных CSV-файла и запрос для предварительной обработки данных с помощью `clickhouse-local`.

Предварительная обработка включает:
- разделение почтового индекса на два разных столбца `postcode1` и `postcode2`, что лучше подходит для хранения данных и выполнения запросов к ним;
- преобразование поля `time` в дату, поскольку оно содержит только время 00:00;
- поле [UUid](../../sql-reference/data-types/uuid.md) игнорируется, потому что оно не будет использовано для анализа;
- преобразование полей `type` и `duration` в более читаемые поля типа `Enum` с помощью функции [transform](../../sql-reference/functions/other-functions.md#transform);
- преобразование полей `is_new` и `category` из односимвольной строки (`Y`/`N` и `A`/`B`) в поле [UInt8](../../sql-reference/data-types/int-uint.md#uint8-uint16-uint32-uint64-uint256-int8-int16-int32-int64-int128-int256) со значениями 0 и 1 соответственно.

Обработанные данные передаются в `clickhouse-client` и импортируются в таблицу ClickHouse потоковым способом.

```bash
clickhouse-local --input-format CSV --structure '
    uuid String,
    price UInt32,
    time DateTime,
    postcode String,
    a String,
    b String,
    c String,
    addr1 String,
    addr2 String,
    street String,
    locality String,
    town String,
    district String,
    county String,
    d String,
    e String
' --query "
    WITH splitByChar(' ', postcode) AS p
    SELECT
        price,
        toDate(time) AS date,
        p[1] AS postcode1,
        p[2] AS postcode2,
        transform(a, ['T', 'S', 'D', 'F', 'O'], ['terraced', 'semi-detached', 'detached', 'flat', 'other']) AS type,
        b = 'Y' AS is_new,
        transform(c, ['F', 'L', 'U'], ['freehold', 'leasehold', 'unknown']) AS duration,
        addr1,
        addr2,
        street,
        locality,
        town,
        district,
        county,
        d = 'B' AS category
    FROM table" --date_time_input_format best_effort < pp-complete.csv | clickhouse-client --query "INSERT INTO uk_price_paid FORMAT TSV"
```

Выполнение запроса займет около 40 секунд.

## Проверьте импортированные данные {#validate-data}

Запрос:

```sql
SELECT count() FROM uk_price_paid;
```

Результат:

```text
┌──count()─┐
│ 26321785 │
└──────────┘
```

Размер набора данных в ClickHouse составляет всего 278 MiB, проверьте это.

Запрос:

```sql
SELECT formatReadableSize(total_bytes) FROM system.tables WHERE name = 'uk_price_paid';
```

Результат:

```text
┌─formatReadableSize(total_bytes)─┐
│ 278.80 MiB                      │
└─────────────────────────────────┘
```

## Примеры запросов {#run-queries}

### Запрос 1. Средняя цена за год {#average-price}

Запрос:

```sql
SELECT toYear(date) AS year, round(avg(price)) AS price, bar(price, 0, 1000000, 80) FROM uk_price_paid GROUP BY year ORDER BY year;
```

Результат:

```text
┌─year─┬──price─┬─bar(round(avg(price)), 0, 1000000, 80)─┐
│ 1995 │  67932 │ █████▍                                 │
│ 1996 │  71505 │ █████▋                                 │
│ 1997 │  78532 │ ██████▎                                │
│ 1998 │  85436 │ ██████▋                                │
│ 1999 │  96037 │ ███████▋                               │
│ 2000 │ 107479 │ ████████▌                              │
│ 2001 │ 118885 │ █████████▌                             │
│ 2002 │ 137941 │ ███████████                            │
│ 2003 │ 155889 │ ████████████▍                          │
│ 2004 │ 178885 │ ██████████████▎                        │
│ 2005 │ 189351 │ ███████████████▏                       │
│ 2006 │ 203528 │ ████████████████▎                      │
│ 2007 │ 219378 │ █████████████████▌                     │
│ 2008 │ 217056 │ █████████████████▎                     │
│ 2009 │ 213419 │ █████████████████                      │
│ 2010 │ 236109 │ ██████████████████▊                    │
│ 2011 │ 232805 │ ██████████████████▌                    │
│ 2012 │ 238367 │ ███████████████████                    │
│ 2013 │ 256931 │ ████████████████████▌                  │
│ 2014 │ 279915 │ ██████████████████████▍                │
│ 2015 │ 297266 │ ███████████████████████▋               │
│ 2016 │ 313201 │ █████████████████████████              │
│ 2017 │ 346097 │ ███████████████████████████▋           │
│ 2018 │ 350116 │ ████████████████████████████           │
│ 2019 │ 351013 │ ████████████████████████████           │
│ 2020 │ 369420 │ █████████████████████████████▌         │
│ 2021 │ 386903 │ ██████████████████████████████▊        │
└──────┴────────┴────────────────────────────────────────┘
```

### Запрос 2. Средняя цена за год в Лондоне {#average-price-london}

Запрос:

```sql
SELECT toYear(date) AS year, round(avg(price)) AS price, bar(price, 0, 2000000, 100) FROM uk_price_paid WHERE town = 'LONDON' GROUP BY year ORDER BY year;
```

Результат:

```text
┌─year─┬───price─┬─bar(round(avg(price)), 0, 2000000, 100)───────────────┐
│ 1995 │  109116 │ █████▍                                                │
│ 1996 │  118667 │ █████▊                                                │
│ 1997 │  136518 │ ██████▋                                               │
│ 1998 │  152983 │ ███████▋                                              │
│ 1999 │  180637 │ █████████                                             │
│ 2000 │  215838 │ ██████████▋                                           │
│ 2001 │  232994 │ ███████████▋                                          │
│ 2002 │  263670 │ █████████████▏                                        │
│ 2003 │  278394 │ █████████████▊                                        │
│ 2004 │  304666 │ ███████████████▏                                      │
│ 2005 │  322875 │ ████████████████▏                                     │
│ 2006 │  356191 │ █████████████████▋                                    │
│ 2007 │  404054 │ ████████████████████▏                                 │
│ 2008 │  420741 │ █████████████████████                                 │
│ 2009 │  427753 │ █████████████████████▍                                │
│ 2010 │  480306 │ ████████████████████████                              │
│ 2011 │  496274 │ ████████████████████████▋                             │
│ 2012 │  519442 │ █████████████████████████▊                            │
│ 2013 │  616212 │ ██████████████████████████████▋                       │
│ 2014 │  724154 │ ████████████████████████████████████▏                 │
│ 2015 │  792129 │ ███████████████████████████████████████▌              │
│ 2016 │  843655 │ ██████████████████████████████████████████▏           │
│ 2017 │  982642 │ █████████████████████████████████████████████████▏    │
│ 2018 │ 1016835 │ ██████████████████████████████████████████████████▋   │
│ 2019 │ 1042849 │ ████████████████████████████████████████████████████▏ │
│ 2020 │ 1011889 │ ██████████████████████████████████████████████████▌   │
│ 2021 │  960343 │ ████████████████████████████████████████████████      │
└──────┴─────────┴───────────────────────────────────────────────────────┘
```

Что-то случилось в 2013 году. Я понятия не имею. Может быть, вы имеете представление о том, что произошло в 2020 году?

### Запрос 3. Самые дорогие районы {#most-expensive-neighborhoods}

Запрос:

```sql
SELECT
    town,
    district,
    count() AS c,
    round(avg(price)) AS price,
    bar(price, 0, 5000000, 100)
FROM uk_price_paid
WHERE date >= '2020-01-01'
GROUP BY
    town,
    district
HAVING c >= 100
ORDER BY price DESC
LIMIT 100;
```

Результат:

```text

┌─town─────────────────┬─district───────────────┬────c─┬───price─┬─bar(round(avg(price)), 0, 5000000, 100)────────────────────────────┐
│ LONDON               │ CITY OF WESTMINSTER    │ 3606 │ 3280239 │ █████████████████████████████████████████████████████████████████▌ │
│ LONDON               │ CITY OF LONDON         │  274 │ 3160502 │ ███████████████████████████████████████████████████████████████▏   │
│ LONDON               │ KENSINGTON AND CHELSEA │ 2550 │ 2308478 │ ██████████████████████████████████████████████▏                    │
│ LEATHERHEAD          │ ELMBRIDGE              │  114 │ 1897407 │ █████████████████████████████████████▊                             │
│ LONDON               │ CAMDEN                 │ 3033 │ 1805404 │ ████████████████████████████████████                               │
│ VIRGINIA WATER       │ RUNNYMEDE              │  156 │ 1753247 │ ███████████████████████████████████                                │
│ WINDLESHAM           │ SURREY HEATH           │  108 │ 1677613 │ █████████████████████████████████▌                                 │
│ THORNTON HEATH       │ CROYDON                │  546 │ 1671721 │ █████████████████████████████████▍                                 │
│ BARNET               │ ENFIELD                │  124 │ 1505840 │ ██████████████████████████████                                     │
│ COBHAM               │ ELMBRIDGE              │  387 │ 1237250 │ ████████████████████████▋                                          │
│ LONDON               │ ISLINGTON              │ 2668 │ 1236980 │ ████████████████████████▋                                          │
│ OXFORD               │ SOUTH OXFORDSHIRE      │  321 │ 1220907 │ ████████████████████████▍                                          │
│ LONDON               │ RICHMOND UPON THAMES   │  704 │ 1215551 │ ████████████████████████▎                                          │
│ LONDON               │ HOUNSLOW               │  671 │ 1207493 │ ████████████████████████▏                                          │
│ ASCOT                │ WINDSOR AND MAIDENHEAD │  407 │ 1183299 │ ███████████████████████▋                                           │
│ BEACONSFIELD         │ BUCKINGHAMSHIRE        │  330 │ 1175615 │ ███████████████████████▌                                           │
│ RICHMOND             │ RICHMOND UPON THAMES   │  874 │ 1110444 │ ██████████████████████▏                                            │
│ LONDON               │ HAMMERSMITH AND FULHAM │ 3086 │ 1053983 │ █████████████████████                                              │
│ SURBITON             │ ELMBRIDGE              │  100 │ 1011800 │ ████████████████████▏                                              │
│ RADLETT              │ HERTSMERE              │  283 │ 1011712 │ ████████████████████▏                                              │
│ SALCOMBE             │ SOUTH HAMS             │  127 │ 1011624 │ ████████████████████▏                                              │
│ WEYBRIDGE            │ ELMBRIDGE              │  655 │ 1007265 │ ████████████████████▏                                              │
│ ESHER                │ ELMBRIDGE              │  485 │  986581 │ ███████████████████▋                                               │
│ LEATHERHEAD          │ GUILDFORD              │  202 │  977320 │ ███████████████████▌                                               │
│ BURFORD              │ WEST OXFORDSHIRE       │  111 │  966893 │ ███████████████████▎                                               │
│ BROCKENHURST         │ NEW FOREST             │  129 │  956675 │ ███████████████████▏                                               │
│ HINDHEAD             │ WAVERLEY               │  137 │  953753 │ ███████████████████                                                │
│ GERRARDS CROSS       │ BUCKINGHAMSHIRE        │  419 │  951121 │ ███████████████████                                                │
│ EAST MOLESEY         │ ELMBRIDGE              │  192 │  936769 │ ██████████████████▋                                                │
│ CHALFONT ST GILES    │ BUCKINGHAMSHIRE        │  146 │  925515 │ ██████████████████▌                                                │
│ LONDON               │ TOWER HAMLETS          │ 4388 │  918304 │ ██████████████████▎                                                │
│ OLNEY                │ MILTON KEYNES          │  235 │  910646 │ ██████████████████▏                                                │
│ HENLEY-ON-THAMES     │ SOUTH OXFORDSHIRE      │  540 │  902418 │ ██████████████████                                                 │
│ LONDON               │ SOUTHWARK              │ 3885 │  892997 │ █████████████████▋                                                 │
│ KINGSTON UPON THAMES │ KINGSTON UPON THAMES   │  960 │  885969 │ █████████████████▋                                                 │
│ LONDON               │ EALING                 │ 2658 │  871755 │ █████████████████▍                                                 │
│ CRANBROOK            │ TUNBRIDGE WELLS        │  431 │  862348 │ █████████████████▏                                                 │
│ LONDON               │ MERTON                 │ 2099 │  859118 │ █████████████████▏                                                 │
│ BELVEDERE            │ BEXLEY                 │  346 │  842423 │ ████████████████▋                                                  │
│ GUILDFORD            │ WAVERLEY               │  143 │  841277 │ ████████████████▋                                                  │
│ HARPENDEN            │ ST ALBANS              │  657 │  841216 │ ████████████████▋                                                  │
│ LONDON               │ HACKNEY                │ 3307 │  837090 │ ████████████████▋                                                  │
│ LONDON               │ WANDSWORTH             │ 6566 │  832663 │ ████████████████▋                                                  │
│ MAIDENHEAD           │ BUCKINGHAMSHIRE        │  123 │  824299 │ ████████████████▍                                                  │
│ KINGS LANGLEY        │ DACORUM                │  145 │  821331 │ ████████████████▍                                                  │
│ BERKHAMSTED          │ DACORUM                │  543 │  818415 │ ████████████████▎                                                  │
│ GREAT MISSENDEN      │ BUCKINGHAMSHIRE        │  226 │  802807 │ ████████████████                                                   │
│ BILLINGSHURST        │ CHICHESTER             │  144 │  797829 │ ███████████████▊                                                   │
│ WOKING               │ GUILDFORD              │  176 │  793494 │ ███████████████▋                                                   │
│ STOCKBRIDGE          │ TEST VALLEY            │  178 │  793269 │ ███████████████▋                                                   │
│ EPSOM                │ REIGATE AND BANSTEAD   │  172 │  791862 │ ███████████████▋                                                   │
│ TONBRIDGE            │ TUNBRIDGE WELLS        │  360 │  787876 │ ███████████████▋                                                   │
│ TEDDINGTON           │ RICHMOND UPON THAMES   │  595 │  786492 │ ███████████████▋                                                   │
│ TWICKENHAM           │ RICHMOND UPON THAMES   │ 1155 │  786193 │ ███████████████▋                                                   │
│ LYNDHURST            │ NEW FOREST             │  102 │  785593 │ ███████████████▋                                                   │
│ LONDON               │ LAMBETH                │ 5228 │  774574 │ ███████████████▍                                                   │
│ LONDON               │ BARNET                 │ 3955 │  773259 │ ███████████████▍                                                   │
│ OXFORD               │ VALE OF WHITE HORSE    │  353 │  772088 │ ███████████████▍                                                   │
│ TONBRIDGE            │ MAIDSTONE              │  305 │  770740 │ ███████████████▍                                                   │
│ LUTTERWORTH          │ HARBOROUGH             │  538 │  768634 │ ███████████████▎                                                   │
│ WOODSTOCK            │ WEST OXFORDSHIRE       │  140 │  766037 │ ███████████████▎                                                   │
│ MIDHURST             │ CHICHESTER             │  257 │  764815 │ ███████████████▎                                                   │
│ MARLOW               │ BUCKINGHAMSHIRE        │  327 │  761876 │ ███████████████▏                                                   │
│ LONDON               │ NEWHAM                 │ 3237 │  761784 │ ███████████████▏                                                   │
│ ALDERLEY EDGE        │ CHESHIRE EAST          │  178 │  757318 │ ███████████████▏                                                   │
│ LUTON                │ CENTRAL BEDFORDSHIRE   │  212 │  754283 │ ███████████████                                                    │
│ PETWORTH             │ CHICHESTER             │  154 │  754220 │ ███████████████                                                    │
│ ALRESFORD            │ WINCHESTER             │  219 │  752718 │ ███████████████                                                    │
│ POTTERS BAR          │ WELWYN HATFIELD        │  174 │  748465 │ ██████████████▊                                                    │
│ HASLEMERE            │ CHICHESTER             │  128 │  746907 │ ██████████████▊                                                    │
│ TADWORTH             │ REIGATE AND BANSTEAD   │  502 │  743252 │ ██████████████▋                                                    │
│ THAMES DITTON        │ ELMBRIDGE              │  244 │  741913 │ ██████████████▋                                                    │
│ REIGATE              │ REIGATE AND BANSTEAD   │  581 │  738198 │ ██████████████▋                                                    │
│ BOURNE END           │ BUCKINGHAMSHIRE        │  138 │  735190 │ ██████████████▋                                                    │
│ SEVENOAKS            │ SEVENOAKS              │ 1156 │  730018 │ ██████████████▌                                                    │
│ OXTED                │ TANDRIDGE              │  336 │  729123 │ ██████████████▌                                                    │
│ INGATESTONE          │ BRENTWOOD              │  166 │  728103 │ ██████████████▌                                                    │
│ LONDON               │ BRENT                  │ 2079 │  720605 │ ██████████████▍                                                    │
│ LONDON               │ HARINGEY               │ 3216 │  717780 │ ██████████████▎                                                    │
│ PURLEY               │ CROYDON                │  575 │  716108 │ ██████████████▎                                                    │
│ WELWYN               │ WELWYN HATFIELD        │  222 │  710603 │ ██████████████▏                                                    │
│ RICKMANSWORTH        │ THREE RIVERS           │  798 │  704571 │ ██████████████                                                     │
│ BANSTEAD             │ REIGATE AND BANSTEAD   │  401 │  701293 │ ██████████████                                                     │
│ CHIGWELL             │ EPPING FOREST          │  261 │  701203 │ ██████████████                                                     │
│ PINNER               │ HARROW                 │  528 │  698885 │ █████████████▊                                                     │
│ HASLEMERE            │ WAVERLEY               │  280 │  696659 │ █████████████▊                                                     │
│ SLOUGH               │ BUCKINGHAMSHIRE        │  396 │  694917 │ █████████████▊                                                     │
│ WALTON-ON-THAMES     │ ELMBRIDGE              │  946 │  692395 │ █████████████▋                                                     │
│ READING              │ SOUTH OXFORDSHIRE      │  318 │  691988 │ █████████████▋                                                     │
│ NORTHWOOD            │ HILLINGDON             │  271 │  690643 │ █████████████▋                                                     │
│ FELTHAM              │ HOUNSLOW               │  763 │  688595 │ █████████████▋                                                     │
│ ASHTEAD              │ MOLE VALLEY            │  303 │  687923 │ █████████████▋                                                     │
│ BARNET               │ BARNET                 │  975 │  686980 │ █████████████▋                                                     │
│ WOKING               │ SURREY HEATH           │  283 │  686669 │ █████████████▋                                                     │
│ MALMESBURY           │ WILTSHIRE              │  323 │  683324 │ █████████████▋                                                     │
│ AMERSHAM             │ BUCKINGHAMSHIRE        │  496 │  680962 │ █████████████▌                                                     │
│ CHISLEHURST          │ BROMLEY                │  430 │  680209 │ █████████████▌                                                     │
│ HYTHE                │ FOLKESTONE AND HYTHE   │  490 │  676908 │ █████████████▌                                                     │
│ MAYFIELD             │ WEALDEN                │  101 │  676210 │ █████████████▌                                                     │
│ ASCOT                │ BRACKNELL FOREST       │  168 │  676004 │ █████████████▌                                                     │
└──────────────────────┴────────────────────────┴──────┴─────────┴────────────────────────────────────────────────────────────────────┘
```

## Ускорьте запросы с помощью проекций {#speedup-with-projections}

[Проекции](../../sql-reference/statements/alter/projection.md) позволяют повысить скорость запросов за счет хранения предварительно агрегированных данных.

### Создайте проекцию {#build-projection}

Создайте агрегирующую проекцию по параметрам `toYear(date)`, `district`, `town`:

```sql
ALTER TABLE uk_price_paid
    ADD PROJECTION projection_by_year_district_town
    (
        SELECT
            toYear(date),
            district,
            town,
            avg(price),
            sum(price),
            count()
        GROUP BY
            toYear(date),
            district,
            town
    );
```

Заполните проекцию для текущих данных (иначе проекция будет создана только для добавляемых данных):

```sql
ALTER TABLE uk_price_paid
    MATERIALIZE PROJECTION projection_by_year_district_town
SETTINGS mutations_sync = 1;
```

## Проверьте производительность {#test-performance}

Давайте выполним те же 3 запроса.

### Запрос 1. Средняя цена за год {#average-price-projections}

Запрос:

```sql
SELECT
    toYear(date) AS year,
    round(avg(price)) AS price,
    bar(price, 0, 1000000, 80)
FROM uk_price_paid
GROUP BY year
ORDER BY year ASC;
```

Результат:

```text
┌─year─┬──price─┬─bar(round(avg(price)), 0, 1000000, 80)─┐
│ 1995 │  67932 │ █████▍                                 │
│ 1996 │  71505 │ █████▋                                 │
│ 1997 │  78532 │ ██████▎                                │
│ 1998 │  85436 │ ██████▋                                │
│ 1999 │  96037 │ ███████▋                               │
│ 2000 │ 107479 │ ████████▌                              │
│ 2001 │ 118885 │ █████████▌                             │
│ 2002 │ 137941 │ ███████████                            │
│ 2003 │ 155889 │ ████████████▍                          │
│ 2004 │ 178885 │ ██████████████▎                        │
│ 2005 │ 189351 │ ███████████████▏                       │
│ 2006 │ 203528 │ ████████████████▎                      │
│ 2007 │ 219378 │ █████████████████▌                     │
│ 2008 │ 217056 │ █████████████████▎                     │
│ 2009 │ 213419 │ █████████████████                      │
│ 2010 │ 236109 │ ██████████████████▊                    │
│ 2011 │ 232805 │ ██████████████████▌                    │
│ 2012 │ 238367 │ ███████████████████                    │
│ 2013 │ 256931 │ ████████████████████▌                  │
│ 2014 │ 279915 │ ██████████████████████▍                │
│ 2015 │ 297266 │ ███████████████████████▋               │
│ 2016 │ 313201 │ █████████████████████████              │
│ 2017 │ 346097 │ ███████████████████████████▋           │
│ 2018 │ 350116 │ ████████████████████████████           │
│ 2019 │ 351013 │ ████████████████████████████           │
│ 2020 │ 369420 │ █████████████████████████████▌         │
│ 2021 │ 386903 │ ██████████████████████████████▊        │
└──────┴────────┴────────────────────────────────────────┘
```

### Запрос 2. Средняя цена за год в Лондоне {#average-price-london-projections}

Запрос:

```sql
SELECT
    toYear(date) AS year,
    round(avg(price)) AS price,
    bar(price, 0, 2000000, 100)
FROM uk_price_paid
WHERE town = 'LONDON'
GROUP BY year
ORDER BY year ASC;
```

Результат:

```text
┌─year─┬───price─┬─bar(round(avg(price)), 0, 2000000, 100)───────────────┐
│ 1995 │  109116 │ █████▍                                                │
│ 1996 │  118667 │ █████▊                                                │
│ 1997 │  136518 │ ██████▋                                               │
│ 1998 │  152983 │ ███████▋                                              │
│ 1999 │  180637 │ █████████                                             │
│ 2000 │  215838 │ ██████████▋                                           │
│ 2001 │  232994 │ ███████████▋                                          │
│ 2002 │  263670 │ █████████████▏                                        │
│ 2003 │  278394 │ █████████████▊                                        │
│ 2004 │  304666 │ ███████████████▏                                      │
│ 2005 │  322875 │ ████████████████▏                                     │
│ 2006 │  356191 │ █████████████████▋                                    │
│ 2007 │  404054 │ ████████████████████▏                                 │
│ 2008 │  420741 │ █████████████████████                                 │
│ 2009 │  427753 │ █████████████████████▍                                │
│ 2010 │  480306 │ ████████████████████████                              │
│ 2011 │  496274 │ ████████████████████████▋                             │
│ 2012 │  519442 │ █████████████████████████▊                            │
│ 2013 │  616212 │ ██████████████████████████████▋                       │
│ 2014 │  724154 │ ████████████████████████████████████▏                 │
│ 2015 │  792129 │ ███████████████████████████████████████▌              │
│ 2016 │  843655 │ ██████████████████████████████████████████▏           │
│ 2017 │  982642 │ █████████████████████████████████████████████████▏    │
│ 2018 │ 1016835 │ ██████████████████████████████████████████████████▋   │
│ 2019 │ 1042849 │ ████████████████████████████████████████████████████▏ │
│ 2020 │ 1011889 │ ██████████████████████████████████████████████████▌   │
│ 2021 │  960343 │ ████████████████████████████████████████████████      │
└──────┴─────────┴───────────────────────────────────────────────────────┘
```

### Запрос 3. Самые дорогие районы {#most-expensive-neighborhoods-projections}

Условие (date >= '2020-01-01') необходимо изменить, чтобы оно соответствовало проекции (toYear(date) >= 2020).

Запрос:

```sql
SELECT
    town,
    district,
    count() AS c,
    round(avg(price)) AS price,
    bar(price, 0, 5000000, 100)
FROM uk_price_paid
WHERE toYear(date) >= 2020
GROUP BY
    town,
    district
HAVING c >= 100
ORDER BY price DESC
LIMIT 100;
```

Результат:

```text
┌─town─────────────────┬─district───────────────┬────c─┬───price─┬─bar(round(avg(price)), 0, 5000000, 100)────────────────────────────┐
│ LONDON               │ CITY OF WESTMINSTER    │ 3606 │ 3280239 │ █████████████████████████████████████████████████████████████████▌ │
│ LONDON               │ CITY OF LONDON         │  274 │ 3160502 │ ███████████████████████████████████████████████████████████████▏   │
│ LONDON               │ KENSINGTON AND CHELSEA │ 2550 │ 2308478 │ ██████████████████████████████████████████████▏                    │
│ LEATHERHEAD          │ ELMBRIDGE              │  114 │ 1897407 │ █████████████████████████████████████▊                             │
│ LONDON               │ CAMDEN                 │ 3033 │ 1805404 │ ████████████████████████████████████                               │
│ VIRGINIA WATER       │ RUNNYMEDE              │  156 │ 1753247 │ ███████████████████████████████████                                │
│ WINDLESHAM           │ SURREY HEATH           │  108 │ 1677613 │ █████████████████████████████████▌                                 │
│ THORNTON HEATH       │ CROYDON                │  546 │ 1671721 │ █████████████████████████████████▍                                 │
│ BARNET               │ ENFIELD                │  124 │ 1505840 │ ██████████████████████████████                                     │
│ COBHAM               │ ELMBRIDGE              │  387 │ 1237250 │ ████████████████████████▋                                          │
│ LONDON               │ ISLINGTON              │ 2668 │ 1236980 │ ████████████████████████▋                                          │
│ OXFORD               │ SOUTH OXFORDSHIRE      │  321 │ 1220907 │ ████████████████████████▍                                          │
│ LONDON               │ RICHMOND UPON THAMES   │  704 │ 1215551 │ ████████████████████████▎                                          │
│ LONDON               │ HOUNSLOW               │  671 │ 1207493 │ ████████████████████████▏                                          │
│ ASCOT                │ WINDSOR AND MAIDENHEAD │  407 │ 1183299 │ ███████████████████████▋                                           │
│ BEACONSFIELD         │ BUCKINGHAMSHIRE        │  330 │ 1175615 │ ███████████████████████▌                                           │
│ RICHMOND             │ RICHMOND UPON THAMES   │  874 │ 1110444 │ ██████████████████████▏                                            │
│ LONDON               │ HAMMERSMITH AND FULHAM │ 3086 │ 1053983 │ █████████████████████                                              │
│ SURBITON             │ ELMBRIDGE              │  100 │ 1011800 │ ████████████████████▏                                              │
│ RADLETT              │ HERTSMERE              │  283 │ 1011712 │ ████████████████████▏                                              │
│ SALCOMBE             │ SOUTH HAMS             │  127 │ 1011624 │ ████████████████████▏                                              │
│ WEYBRIDGE            │ ELMBRIDGE              │  655 │ 1007265 │ ████████████████████▏                                              │
│ ESHER                │ ELMBRIDGE              │  485 │  986581 │ ███████████████████▋                                               │
│ LEATHERHEAD          │ GUILDFORD              │  202 │  977320 │ ███████████████████▌                                               │
│ BURFORD              │ WEST OXFORDSHIRE       │  111 │  966893 │ ███████████████████▎                                               │
│ BROCKENHURST         │ NEW FOREST             │  129 │  956675 │ ███████████████████▏                                               │
│ HINDHEAD             │ WAVERLEY               │  137 │  953753 │ ███████████████████                                                │
│ GERRARDS CROSS       │ BUCKINGHAMSHIRE        │  419 │  951121 │ ███████████████████                                                │
│ EAST MOLESEY         │ ELMBRIDGE              │  192 │  936769 │ ██████████████████▋                                                │
│ CHALFONT ST GILES    │ BUCKINGHAMSHIRE        │  146 │  925515 │ ██████████████████▌                                                │
│ LONDON               │ TOWER HAMLETS          │ 4388 │  918304 │ ██████████████████▎                                                │
│ OLNEY                │ MILTON KEYNES          │  235 │  910646 │ ██████████████████▏                                                │
│ HENLEY-ON-THAMES     │ SOUTH OXFORDSHIRE      │  540 │  902418 │ ██████████████████                                                 │
│ LONDON               │ SOUTHWARK              │ 3885 │  892997 │ █████████████████▋                                                 │
│ KINGSTON UPON THAMES │ KINGSTON UPON THAMES   │  960 │  885969 │ █████████████████▋                                                 │
│ LONDON               │ EALING                 │ 2658 │  871755 │ █████████████████▍                                                 │
│ CRANBROOK            │ TUNBRIDGE WELLS        │  431 │  862348 │ █████████████████▏                                                 │
│ LONDON               │ MERTON                 │ 2099 │  859118 │ █████████████████▏                                                 │
│ BELVEDERE            │ BEXLEY                 │  346 │  842423 │ ████████████████▋                                                  │
│ GUILDFORD            │ WAVERLEY               │  143 │  841277 │ ████████████████▋                                                  │
│ HARPENDEN            │ ST ALBANS              │  657 │  841216 │ ████████████████▋                                                  │
│ LONDON               │ HACKNEY                │ 3307 │  837090 │ ████████████████▋                                                  │
│ LONDON               │ WANDSWORTH             │ 6566 │  832663 │ ████████████████▋                                                  │
│ MAIDENHEAD           │ BUCKINGHAMSHIRE        │  123 │  824299 │ ████████████████▍                                                  │
│ KINGS LANGLEY        │ DACORUM                │  145 │  821331 │ ████████████████▍                                                  │
│ BERKHAMSTED          │ DACORUM                │  543 │  818415 │ ████████████████▎                                                  │
│ GREAT MISSENDEN      │ BUCKINGHAMSHIRE        │  226 │  802807 │ ████████████████                                                   │
│ BILLINGSHURST        │ CHICHESTER             │  144 │  797829 │ ███████████████▊                                                   │
│ WOKING               │ GUILDFORD              │  176 │  793494 │ ███████████████▋                                                   │
│ STOCKBRIDGE          │ TEST VALLEY            │  178 │  793269 │ ███████████████▋                                                   │
│ EPSOM                │ REIGATE AND BANSTEAD   │  172 │  791862 │ ███████████████▋                                                   │
│ TONBRIDGE            │ TUNBRIDGE WELLS        │  360 │  787876 │ ███████████████▋                                                   │
│ TEDDINGTON           │ RICHMOND UPON THAMES   │  595 │  786492 │ ███████████████▋                                                   │
│ TWICKENHAM           │ RICHMOND UPON THAMES   │ 1155 │  786193 │ ███████████████▋                                                   │
│ LYNDHURST            │ NEW FOREST             │  102 │  785593 │ ███████████████▋                                                   │
│ LONDON               │ LAMBETH                │ 5228 │  774574 │ ███████████████▍                                                   │
│ LONDON               │ BARNET                 │ 3955 │  773259 │ ███████████████▍                                                   │
│ OXFORD               │ VALE OF WHITE HORSE    │  353 │  772088 │ ███████████████▍                                                   │
│ TONBRIDGE            │ MAIDSTONE              │  305 │  770740 │ ███████████████▍                                                   │
│ LUTTERWORTH          │ HARBOROUGH             │  538 │  768634 │ ███████████████▎                                                   │
│ WOODSTOCK            │ WEST OXFORDSHIRE       │  140 │  766037 │ ███████████████▎                                                   │
│ MIDHURST             │ CHICHESTER             │  257 │  764815 │ ███████████████▎                                                   │
│ MARLOW               │ BUCKINGHAMSHIRE        │  327 │  761876 │ ███████████████▏                                                   │
│ LONDON               │ NEWHAM                 │ 3237 │  761784 │ ███████████████▏                                                   │
│ ALDERLEY EDGE        │ CHESHIRE EAST          │  178 │  757318 │ ███████████████▏                                                   │
│ LUTON                │ CENTRAL BEDFORDSHIRE   │  212 │  754283 │ ███████████████                                                    │
│ PETWORTH             │ CHICHESTER             │  154 │  754220 │ ███████████████                                                    │
│ ALRESFORD            │ WINCHESTER             │  219 │  752718 │ ███████████████                                                    │
│ POTTERS BAR          │ WELWYN HATFIELD        │  174 │  748465 │ ██████████████▊                                                    │
│ HASLEMERE            │ CHICHESTER             │  128 │  746907 │ ██████████████▊                                                    │
│ TADWORTH             │ REIGATE AND BANSTEAD   │  502 │  743252 │ ██████████████▋                                                    │
│ THAMES DITTON        │ ELMBRIDGE              │  244 │  741913 │ ██████████████▋                                                    │
│ REIGATE              │ REIGATE AND BANSTEAD   │  581 │  738198 │ ██████████████▋                                                    │
│ BOURNE END           │ BUCKINGHAMSHIRE        │  138 │  735190 │ ██████████████▋                                                    │
│ SEVENOAKS            │ SEVENOAKS              │ 1156 │  730018 │ ██████████████▌                                                    │
│ OXTED                │ TANDRIDGE              │  336 │  729123 │ ██████████████▌                                                    │
│ INGATESTONE          │ BRENTWOOD              │  166 │  728103 │ ██████████████▌                                                    │
│ LONDON               │ BRENT                  │ 2079 │  720605 │ ██████████████▍                                                    │
│ LONDON               │ HARINGEY               │ 3216 │  717780 │ ██████████████▎                                                    │
│ PURLEY               │ CROYDON                │  575 │  716108 │ ██████████████▎                                                    │
│ WELWYN               │ WELWYN HATFIELD        │  222 │  710603 │ ██████████████▏                                                    │
│ RICKMANSWORTH        │ THREE RIVERS           │  798 │  704571 │ ██████████████                                                     │
│ BANSTEAD             │ REIGATE AND BANSTEAD   │  401 │  701293 │ ██████████████                                                     │
│ CHIGWELL             │ EPPING FOREST          │  261 │  701203 │ ██████████████                                                     │
│ PINNER               │ HARROW                 │  528 │  698885 │ █████████████▊                                                     │
│ HASLEMERE            │ WAVERLEY               │  280 │  696659 │ █████████████▊                                                     │
│ SLOUGH               │ BUCKINGHAMSHIRE        │  396 │  694917 │ █████████████▊                                                     │
│ WALTON-ON-THAMES     │ ELMBRIDGE              │  946 │  692395 │ █████████████▋                                                     │
│ READING              │ SOUTH OXFORDSHIRE      │  318 │  691988 │ █████████████▋                                                     │
│ NORTHWOOD            │ HILLINGDON             │  271 │  690643 │ █████████████▋                                                     │
│ FELTHAM              │ HOUNSLOW               │  763 │  688595 │ █████████████▋                                                     │
│ ASHTEAD              │ MOLE VALLEY            │  303 │  687923 │ █████████████▋                                                     │
│ BARNET               │ BARNET                 │  975 │  686980 │ █████████████▋                                                     │
│ WOKING               │ SURREY HEATH           │  283 │  686669 │ █████████████▋                                                     │
│ MALMESBURY           │ WILTSHIRE              │  323 │  683324 │ █████████████▋                                                     │
│ AMERSHAM             │ BUCKINGHAMSHIRE        │  496 │  680962 │ █████████████▌                                                     │
│ CHISLEHURST          │ BROMLEY                │  430 │  680209 │ █████████████▌                                                     │
│ HYTHE                │ FOLKESTONE AND HYTHE   │  490 │  676908 │ █████████████▌                                                     │
│ MAYFIELD             │ WEALDEN                │  101 │  676210 │ █████████████▌                                                     │
│ ASCOT                │ BRACKNELL FOREST       │  168 │  676004 │ █████████████▌                                                     │
└──────────────────────┴────────────────────────┴──────┴─────────┴────────────────────────────────────────────────────────────────────┘
```

### Резюме {#summary}

Все три запроса работают намного быстрее и читают меньшее количество строк.

```text
Query 1

no projection: 27 rows in set. Elapsed: 0.158 sec. Processed 26.32 million rows, 157.93 MB (166.57 million rows/s., 999.39 MB/s.)
   projection: 27 rows in set. Elapsed: 0.007 sec. Processed 105.96 thousand rows, 3.33 MB (14.58 million rows/s., 458.13 MB/s.)


Query 2

no projection: 27 rows in set. Elapsed: 0.163 sec. Processed 26.32 million rows, 80.01 MB (161.75 million rows/s., 491.64 MB/s.)
   projection: 27 rows in set. Elapsed: 0.008 sec. Processed 105.96 thousand rows, 3.67 MB (13.29 million rows/s., 459.89 MB/s.)

Query 3

no projection: 100 rows in set. Elapsed: 0.069 sec. Processed 26.32 million rows, 62.47 MB (382.13 million rows/s., 906.93 MB/s.)
   projection: 100 rows in set. Elapsed: 0.029 sec. Processed 8.08 thousand rows, 511.08 KB (276.06 thousand rows/s., 17.47 MB/s.)
```

### Online Playground {#playground}

Этот набор данных доступен в [Online Playground](https://gh-api.clickhouse.tech/play?user=play#U0VMRUNUIHRvd24sIGRpc3RyaWN0LCBjb3VudCgpIEFTIGMsIHJvdW5kKGF2ZyhwcmljZSkpIEFTIHByaWNlLCBiYXIocHJpY2UsIDAsIDUwMDAwMDAsIDEwMCkgRlJPTSB1a19wcmljZV9wYWlkIFdIRVJFIGRhdGUgPj0gJzIwMjAtMDEtMDEnIEdST1VQIEJZIHRvd24sIGRpc3RyaWN0IEhBVklORyBjID49IDEwMCBPUkRFUiBCWSBwcmljZSBERVNDIExJTUlUIDEwMA==).
