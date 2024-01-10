# Обработка паттернов в данных

**Паттерн данных** – это комбинация событий, условий и корреляций между этими событиями для отслеживания различных закономерностей и выявления событий. Поиск паттернов используются для анализа и мониторинга потока событий в реальном времени, что позволяет оперативно реагировать на изменения и принимать важные решения. В системах анализа данных паттерн данных формулирует правило, по которому система определяет, соответствует ли входящий поток событий определенным критериям, приводя к срабатыванию определенных действий или уведомлений.

Разберем практический пример обработки паттернов в потоке данных, создаваемым  IoT-устройством с кнопками и событиями, вызываемыми при их активации. Нам нужно найти и обработать следующую последовательность нажатия кнопок: «кнопка 1», «кнопка 2», «кнопка 3». Данные передаются в виде JSON-строк, которые с помощью [привязок к данным](glossary.md#binding) распределяются по колонкам `ts` и `button` потока данных `input_stream`.

Структура передаваемых данных:
```json
{"ts":1700000000, "button": 1, "device_id": 1, "zone_id": 24}
{"ts":1701000000, "button": 2, "device_id": 2, "zone_id": 12}
```

Тело SQL-запроса к {{ yql-short-name }}:

```sql
SELECT * FROM input_stream MATCH_RECOGNIZE ( -- выполняем паттерн-матчинг из потока input_stream
    ORDER BY ts -- просматриваем события в порядке возрастания значения колонки ts (тип данных Timestamp)
    MEASURES
      LAST(B1.ts) AS b1, -- в результатах запроса будем получать последний момент нажатия на кнопку 1
      LAST(B3.ts) AS b3  -- в результатах запроса будем получать последний момент нажатия на кнопку 3

    ONE ROW PER MATCH    -- будем получать одну строку результатов на найденное совпадение
    AFTER MATCH SKIP TO NEXT ROW -- после обнаружения паттерна перейдем на следующую строку
    PATTERN (
      B1 B2+ B3 -- Ищем паттерн в данных, состоящий из одного нажатия на кнопку 1, одного или нескольких нажатий на кнопку 2 и одного нажатия на кнопку 3
    )
    DEFINE
        B1 AS B1.button = 1, -- определяем условие B1 как нажатие кнопки 1 (значение поля button равно 1)
        B2 AS B2.button = 2, -- определяем условие B2 как нажатие кнопки 2 (значение поля button равно 2)
        B3 AS B3.button = 3  -- определяем условие B3 как нажатие кнопки 3 (значение поля button равно 3)
) AS MATCHED;
```


## Синтаксис {#syntax}

Команда `MATCH_RECOGNIZE` выполняет поиск данных по заданному паттерну и возвращает найденные результаты. SQL-синтаксис команды `MATCH_RECOGNIZE`:
```sql
MATCH_RECOGNIZE (
    [ PARTITION BY <partition1> [ ... , <partitionN>] ]
    ORDER BY orderItem1 [ ... , orderItemN] [ASC]
    MEASURES LAST(measure_expr1>)|FIRST(measure_expr1>) [AS] <alias1> [ ... , LAST(measure_exprN>)|FIRST(measure_exprN>) [AS] <aliasN>]
    ONE ROW PER MATCH
    [AFTER MATCH SKIP TO NEXT ROW]
    PATTERN (<row_pattern>)
    DEFINE <symbol1> AS <expr1>[ ... , <symbolN> AS <exprN>]
)
```

Описание элементов SQL-синтаксиса команды `MATCH_RECOGNIZE`:
* [`DEFINE`](#define) – условия, которым должны соответствовать строки для каждой из переменных `<symbol1> AS <expr1>[ ... , <symbolN> AS <exprN>]`.
* [`PATTERN`](#pattern) – шаблон для поиска данных. Состоит из переменных и правил поиска паттерна, описанного в выражении `<row_pattern>`. `PATTERN` работает схожим образом с [регулярными выражениями](https://ru.wikipedia.org/wiki/Регулярные_выражения).
* [`ONE ROW PER MATCH`](#rows_per_match) определяет объем выходных данных по каждому найденному совпадению.
* [`AFTER MATCH SKIP TO NEXT ROW`](#after_match_skip_to_next_row) определяет способ перехода к месту поиска следующего совпадения.
* [`MEASURES`](#measures) определяет список выходных столбцов. Каждый столбец из списка `<expr1> [AS] <alias1> [ ... , <exprN> [AS] <aliasN>]` – отдельная конструкция, задающая выходные колонки и описывающая выражения для их вычисления.
* [`ORDER BY`](#order_by) определяет сортировку входных данных. Поиск паттернов выполняется внутри данных, упорядоченных в соответствии со списком колонок или выражениями, перечисленными в `<expr1> [AS] <alias1> [ ... , <exprN> [AS] <aliasN>]`.
* [`PARTITION BY`](#partition_by) разделяет входной поток данных по заданным правилам в соответствии с `<partition1> [ ... , <partitionN>]`. В каждой из частей поиск паттернов производится независимо.


### DEFINE {#define}

```sql
DEFINE <symbol1> AS <expr1>[, <symbol2> AS <expr2>, ...]
```

С помощью `DEFINE` объявляются переменные, которые ищутся во входных данных. Переменные – это имена SQL-выражений, вычисляющихся поверх входных данных. По смыслу SQL-выражения в `DEFINE` совпадают с поисковыми выражениями SQL-конструкции `WHERE`. Например, выражение `button = 1` выполняет поиск всех строк, содержащих колонку `button` со значением `1`. В качестве условий могут выступать любые SQL-выражения, с помощью которых можно выполнять поиск, включая функции агригации (`LAST`, `FIRST`). Например, `button > 2 AND zone_id < 12` или `LAST(button) > 10`.

В SQL-выражениях обязательно нужно указывать название переменной, для которой производится поиск совпадений. Например в следующей SQL-команде, для условия `button = 1` необходимо указать название переменной, для которой производится вычисление - `A`.

```sql
DEFINE
  A AS A.button = 1
```


{% include [info](../_includes/match_recognize_common_exclusion.md) %}


При обработке очередной строки данных производится вычисление всех логических выражений всех переменных ключевого слова `DEFINE`. Если при вычислении выражений переменных `DEFINE` окажется, что логическое выражение принимает значение `ИСТИНА` (`TRUE`), то такая строка получает метку с названием переменной `DEFINE` и добавляется к списку рассматриваемых на совпадение с паттернами.


#### **Примеры** {#define-example}

При описании переменных в SQL-выражениях можно использовать ссылки на другие переменные:

```sql
DEFINE
  A AS A.button = 1,
  B AS B.button > 2 AND A.zone_id < 12
```

Пример SQL-выражения со ссылками на другие переменные:
```sql
DEFINE
  A AS A.button = 1,
  B AS A.button > 2 AND A.device_id = 12 OR A.zone_id < 12
```


### PATTERN {#pattern}

```sql
PATTERN (<row_pattern>)
```

Ключевое слово `PATTERN` описывает поисковый паттерн данных в виде, вычисляющийся на основе переменных из блока `DEFINE`. Синтаксис `PATTERN` схож с синтаксисом [регулярных выражений](https://ru.wikipedia.org/wiki/Регулярные_выражения).

{% note warning %}

Если переменная, использованная в блоке `PATTERN` не была предварительно описана в блоке `DEFINE`, то считается, что она всегда принимает значение `TRUE`.

{% endnote %}

В `PATTERN` можно использовать [квантификаторы](https://ru.wikipedia.org/wiki/Регулярные_выражения#Квантификация_(поиск_последовательностей)). Они в регулярных выражениях определяют число повторений элемента или подпоследовательности в паттерне для нахождения совпадения. Мы будем использовать переменные `A`, `B`, `C` и `D` из блока `DEFINE` для описания квантификаторов. Приведем перечень поддерживаемых квантификаторов:

|Квантификатор|Описание|
|----|-----|
|`A+`|Одно или несколько повторений переменной `A`|
|`A*`|Ноль или несколько повторений переменной `A`|
|`A?`|Ноль или одно повторение переменной `A`|
|`B{n}`|Ровно `n` повторений переменной `B`|
|`C{n, m}`|От `n` до `m` повторений переменной `C`, `m` включительно|
|`D{n,}`|Не менее `n` повторений переменной `D`|
|`(A\|B)`|Появление переменной `A` или `B` в данных|
|`(A\|B){,m}`|Не более `m` повторений переменной `A` или `B`, `m` раз включительно|


Поддерживаемые последовательности поиска паттернов:

|Поддерживаемые последовательности|Синтаксис|Описание|
|---|---|----|
|Последовательность|`A B+ C+ D+`|Производится поиск точной указанной последовательности, не допускается появление других переменных внутри последовательности. Поиск паттерна производится в порядке указания переменных паттерна.|
|Один из|`A \| B \| C`|Переменные перечисляются в любом порядке с указанием символа \| между ними. Производится поиск любой переменной из указанного списка.|
|Группировка|`(A \| B)+ \| C`|Переменные внутри круглых скобок считаются единой группой. В этом случае квантификаторы применяются ко всей группе сразу.|


#### **Пример** {#pattern-example}

```sql
PATTERN (
  B1 E* B2+ B3
)
DEFINE
    B1 as B1.button = 1,
    B2 as B2.button = 2,
    B3 as B3.button = 3
```

В блоке `DEFINE` описаны переменные `B1`, `B2`, `B3`. Переменная `E` в блоке `DEFINE` не описана. Такая запись позволяет интерпретировать `E` как любое событие, поэтому будет искаться следующий паттерн: одно нажатие `кнопки 1`, одно или более нажатий `кнопки 2` и одно нажатие `кнопки 3`. При этом между нажатием `кнопки 1` и `кнопки 2` может быть произвольное число любых других событий.


### ROWS PER MATCH {#rows_per_match}

`ROWS PER MATCH` определяет количество найденных паттернов. `ONE ROW PER MATCH` устанавливает режим работы `ROWS PER MATCH` на вывод одной строки на найденный паттерн. Схемой данных результата будет являться объединение [колонок партиционирования](#partition_by) и всех колонок [измерений](#measures).



### AFTER MATCH SKIP TO NEXT ROW {#after_match_skip_to_next_row}

`AFTER MATCH SKIP TO NEXT ROW` – это опциональный параметр. Он определяет способ перехода от найденного совпадения к поиску следующего совпадения.

Если [паттерн](#pattern) с квантификатором, который подразумевает повторение переменных, указан первым или последним – в результатах поиска возвращается множество строк, каждая из которых соответствует нахождению переменной.


#### Примеры {#match-examples}

Входными данными для всех примеров являются:
```
 button: 1
 button: 1
 button: 2
 button: 2
 button: 3
 button: 3
 button: 3
 ```
 

##### **Пример 1** {#match-example1}

Если квантификатор, допускающий только одно совпадение, в выражении указан первым и последним – возвращается одна строка с результатом.

```sql
MEASURES
  LAST(B1.button) AS b1,
  LAST(B2.button) AS b2,
  LAST(B3.button) AS b3
PATTERN (
  B1 B2+ B3
)
DEFINE
    B1 AS B1.button = 1,
    B2 AS B2.button = 2,
    B3 AS B3.button = 3,

```

Результат:
|b1|b2|b3|
|--|--|--|
|1|2|3|


##### **Пример 2** {#match-example2}

Когда к первой переменной паттерна применен квантификатор, допускающий несколько совпадений, в результат будет добавлено по одной строке на каждое совпадение этого элемента. Таким образом, число результирующих строк будет равно числу совпадений первого элемента паттерна.

```sql
MEASURES
  LAST(B1.button) AS b1,
  LAST(B2.button) AS b2,
  LAST(B3.button) AS b3
PATTERN (
  B1+ B2+ B3
)
DEFINE
    B1 AS B1.button = 1,
    B2 AS B2.button = 2,
    B3 AS B3.button = 3
```

Результат:
|b1|b2|b3|
|--|--|--|
|1|2|3|
|1|2|3|


##### **Пример 3** {#match-example3}

При указании последнего квантификатора, допускающего несколько совпадений, будет возвращено множество строк в соответствии с числом найденных совпадений – по одной строке на каждое из совпадений в начале и по одной строке на каждое совпадение в конце. Результатом будет декартово произведение входных и выходных совпадений.

```sql
MEASURES
  LAST(B1.button) AS b1,
  LAST(B2.button) AS b2,
  LAST(B3.button) AS b3
PATTERN (
  B1 B2+ B3+
)
DEFINE
    B1 AS B1.button = 1,
    B2 AS B2.button = 2,
    B3 AS B3.button = 3
```

Результат:
|b1|b2|b3|
|--|--|--|
|1|2|3|
|1|2|3|
|1|2|3|


##### **Пример 4** {#match-example3}

При указании первого и последнего квантификатора, допускающего несколько совпадений, будет возвращено множество строк в соответствии с числом найденных совпадений.

```sql
MEASURES
  LAST(B1.button) AS b1,
  LAST(B2.button) AS b2,
  LAST(B3.button) AS b3
PATTERN (
  B1+ B2+ B3+
)
DEFINE
    B1 AS B1.button = 1,
    B2 AS B2.button = 2,
    B3 AS B3.button = 3
```

Результат:
|b1|b2|b3|
|--|--|--|
|1|2|3|
|1|2|3|
|1|2|3|
|1|2|3|
|1|2|3|
|1|2|3|


### MEASURES {#measures}

```sql
MEASURES LAST(measure_expr1>)|FIRST(measure_expr1>) [AS] <alias1> [ ... , LAST(measure_exprN>)|FIRST(measure_exprN>) [AS] <aliasN>]
```

`MEASURES` описывает набор возвращаемых колонок при нахождении паттерна. Набор возвращаемых колонок должен быть представлен SQL-выражением c агрегирующими функциями `LAST`/`FIRST` над переменными, объявленными в конструкции [`DEFINE`](#define).


##### **Пример** {#measures-example}

```sql
MEASURES
  LAST(B1.ts) AS b1,
  FIRST(B3.ts) AS b3
```


### ORDER BY {#order_by}

```sql
ORDER BY orderItem1 [ ... , orderItemN] [ASC]

orderItem ::= { <column_alias> | <expr> }
```

`ORDER BY` определяет сортировку входных данных, то есть перед выполнением всех операций поиска паттернов, данные будут предварительно отсортированы по указанным ключам или выражениям. Синтаксис аналогичен SQL-выражению `ORDER BY`. Сортировка допустима только по полям с типом [TimeStamp](https://ydb.tech/ru/docs/yql/reference/types/primitive#datetime), поддерживается только в направлении возрастания, `ASC`.

##### **Пример** {#order_by-example}

```sql
ORDER BY CAST(ts AS Timestamp),
         button_index DESC,
         ts2 ASC
```


### PARTITION BY {#partition_by}

```sql
PARTITION BY <partition1> [ ... , <partitionN>]

partition ::= { <column_alias> | <expr> }
```

`PARTITION BY` – опциональное выражение. `PARTITION BY` партиционирует входные данные по списку полей, указанных в этом ключевом слове, превращая исходные данные в несколько независимых групп потоков, в каждом из которых независимо производится поиск паттернов. Если команда не указывается, то все данные обрабатываются в виде единой группы.

Синтаксис аналогичен SQL-выражению `PARTITION BY` для оконных функций, включая все виды агрегационных функций.

#### **Пример** {#partition_by-example}

```sql
PARTITION BY building
```


## Ограничения {#limitations}

Степень поддержки команды `MATCH_RECOGNIZE` со временем будет соответствовать стандарту [SQL-2016](https://ru.wikipedia.org/wiki/SQL:2016), однако сейчас действует следующий набор ограничений:
- [`ORDER_BY`](#order_by). В качестве столбцов сортировки можно задавать ровно один столбец(выражение) с типом [`Timestamp`](https://ydb.tech/ru/docs/yql/reference/types/primitive#datetime). Сортировка допустима только в направлении возрастания значений, `ASC`.
- [`MEASURES`](#measures). Не поддерживаются агрегационные функции, не поддерживаются функции `PREV`/`NEXT`.
- [`ROWS PER MATCH`](#rows_per_match). Поддерживается только режим `ONE ROW PER MATCH`, режим `ALL ROWS PER MATCH` не поддерживается.
- [`PATTERN`](#pattern). Не реализованы Union pattern variables.
- [`DEFINE`](#define). Не поддерживаются агрегационные функции.