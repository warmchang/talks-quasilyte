# Контроль консистентности кода в Go

![](https://habrastorage.org/webt/t6/m9/bl/t6m9blfwqfwppvqsw6pwotg6nqe.png)

Если вы считаете консистентность важной составляющей качественного кода - эта статья для вас.

Вас ждут:
- Разные способы сделать одно и то же в Go (эквивалентные операции)
- Менее очевидные факторы, влияющие на однородность вашего кода
- Способы увеличения консистентности вашего проекта

<cut/>

# Консистентность - наше что-то

Для начала определимся с тем, что мы называем "консистентностью".

Чем больше исходные коды программы выглядят так, будто их написал один человек, тем более они консистентны.

Стоит заметить, что зачастую даже один и тот же человек может менять свои предпочтения с течением времени, однако по-настоящему остро проблема консистентности встаёт в крупных проектах, где вовлечено большое количество разработчиков.

> Иногда вместо слова "консистентность" используется "согласованность". В этой статье я иногда буду употреблять контекстные синонимы во избежание частой тавтологии.

Существуют разные уровни консистентности, например мы можем выделить три наиболее очевидные:
- Согласованность на уровне одного файла исходного кода
- Согласованность на уровне пакета (или библиотеки)
- Согласованность на уровне всего проекта (если он контролируется одним вендором)

Чем ниже по списку, тем сложнее соблюдать консистентность. При этом отсутствие консистентности на уровне одного файла исходного кода выглядит наиболее отталкивающе.

Можно также спускаться от файла к уровню функции или одного statement, но это, в нашем случае, уже лишняя детализация. Ближе к концу статьи станет понятно почему.

# Эквивалентные операции в Go

В Go не так много идентичных операций, имеющих разное написание (синтаксическое различие), но всё же простор для разногласий найдётся. Какому-то из разработчиков нравится вариант `A`, в то время как второму по вкусу `B`. Оба варианта валидны и имеют своих сторонников. Использование любой формы операции допустимо и не является ошибкой, но использование более, чем одной, формы, может вредить консистентности кода.

Как вы думаете, какой из этих двух способов создать слайс длины 100 используется большинством Go программистов?

```go
// (A)
new([100]T)[:]

// (B)
(&[100]T{})[:]
```

<spoiler title="Ответ"><hr>
Ни один из вариантов не является предпочтительным. В реальном коде я ни разу не видел использования ни одного из них.

А использовать в этом случае стоит `make([]T, 100)`.
<hr></spoiler>

## Одиночный import

Импортирование единственного пакета можно выполнить двумя способами:

```go
// (A) без скобочек
import "github.com/go-lintpack/lintpack"

// (B) со скобочками
import (
    "github.com/go-lintpack/lintpack"
)
```

При этом ни `gofmt`, ни `goimports`, не выполняют преобразование из одной формы в другую. Скорее всего, в вашем проекте встречаются оба варианта.

## Выделение указателя под нулевое значение

Пока в Go есть встроенная функция `new` и альтернативные способы получить указатель на новый объект, вы будете встречать как `new(T)`, так и `&T{}`.

```go
// (A) использование функции new
new(T)
new([]T)

// (B) взятие адреса от литерала
&T{}
&[]T{}
```

## Создание пустого слайса

Для создания пустого слайса (не путать с nil-слайсом) есть как минимум два популярных способа:

```go
// (A) использование функции make
make([]T, 0)

// (A) литерал слайса
[]T{}
```

## Создание пустой хеш-таблицы

Возможно, вам покажется разделение на создание пустого слайса и `map` не очень логичным, но не все люди, которые предпочитают `[]T{}`, будут использовать `map[K]V{}` вместо `make(map[K]V)`. Соответственно, разграничение здесь как минимум не избыточно.

```go
// (A) использование функции make
make(map[K]V)

// (B) литерал хеш-таблицы
map[K]V{}
```

## Шестнадцатеричные литералы

```go
// (A) a-f, нижний регистр
0xff

// (B) A-F, верхний регистр
0xFF
```

Написание типа `0xFf`, со смешенным регистром - это уже не о консистентности. Такое должно быть найдено статическим анализатором (линтером). Каким? Попробуйте, например, [gocritic](https://github.com/go-critic/go-critic).

## Проверка на вхождение в диапазон

В математике (и некоторых языках программирования, но не в Go) вы могли бы описать диапазон как `low < x < high`. Исходный код, который будет выражать это ограничение так записан быть не может. При этом есть как минимум два популярных способа оформить проверку вхождения в диапазон:

```go
// (A) выравнивание по левой стороне
x > low && x < high

// (B) выравнивание по центру
low < x && x < high
```

## Оператор and-not

Вы знали, что в Go есть бинарный оператор `&^`? Называется он `and-not` и выполняет он ту же операцию, что и `&`, применённый к результату `^` от правого (второго) операнда.

```go
// (A) использование оператора &^ (нет пробела)
x &^ y

// (B) использование & с ^ (есть пробел)
x & ^y
```

Я проводил опрос, чтобы убедиться, что здесь вообще могут быть разные предпочтения. В конце концов, если выбор в пользу `&^` был бы единогласным, то это должно было бы быть проверкой линтера, а не частью выбора в вопросе согласованности кода. К моему удивлению, сторонники нашлись у обеих форм.

## Литералы вещественных чисел

Есть множество способов записать вещественный литерал, но одна из самых часто встречающихся особенностей, которая может ломать консистентность даже внутри одной функции - это стиль записи целой и вещественной части (сокращённый или полный).

```go
// (A) явная целая и вещественная части
0.0
1.0

// (B) неявная целая и вещественная части (краткая запись)
.0
1.
```

## LABEL или label?

К сожалению, устоявшихся конвенций к именованию меток, нет. Всё, что остаётся, выбрать один из возможных способов и придерживаться ему.

```go
// (A) всё в верхнем регистре
LABEL_NAME:
    goto PLEASE

// (B) upper camel case
LabelName:
    goto TryTo

// (C) lower camel case
labelName:
    goto beConsistent
```

Возможен также `snake_case`, но нигде, кроме Go ассемблера, я таких меток не видел. Скорее всего, такого варианта придерживаться не стоит.

## Указание типа для untyped числового литерала

```go
// (A) тип находится слева от "="
var x int32 = 10
const y float32 = 1.6

// (B) тип находится справа от "="
var x = int32(10)
const y = float32(1.6)
```

## Перенос закрывающей скобки вызываемой функции

В случае простейших вызовов, которые умещаются на одной строке кода, проблем со скобочками быть не может. Когда по тем или иным причинам вызов функции или метода расползается на несколько строк, появляются несколько степеней свободы, например, вам нужно будет определиться, куда ставить закрывающую список аргументов скобку.

```go
// (A) закрывающая скобка на одной строке с последним аргументом
multiLineCall(
	a,
	b,
	c)

// (B) закрывающая скобка переносится на новую строку
multiLineCall(
	a,
	b,
	c,
)
```

## Проверка на ненулевую длину

```go
// (A) сравнение "количество элементов не равно 0"
len(xs) != 0

// (B) сравнение "более 0 элементов"
len(xs) > 0

// (C) сравнение "не менее 1 элемента"
len(xs) >= 1
```

Для строк обычно используется `s != ""` или `s == ""` ([источник](https://dmitri.shuralyov.com/idiomatic-go#empty-string-check)).

## Расположение default метки в switch

Разумных выборов два: ставить `default` первой или последней меткой. Остальные варианты, вроде "где-то посередине" - это работа для линтера. Проверка `defaultCaseOrder` из `gocritic` находит поможет прийти к более идиоматичному варианту, а `go-consistent` предложит из двух возможных вариантов тот, что сделает код более единообразным.

```go
// (A) default первой меткой
switch {
default:
	return "?"
case x > 10:
	return "more than 10"
}

// (B) default последней меткой
switch {
case x > 10:
	return "more than 10"
default:
	return "?"
}
```

# [go-consistent](https://github.com/Quasilyte/go-consistent)

Выше мы перечислили список эквивалентных операций.

Как определить, какую из них нужно использовать? Наиболее простой вариант ответа: ту, которая имеет более высокую частоту использования в рассматриваемой части проекта (как частный случай, во всём проекте).

Программа [go-consistent](https://github.com/Quasilyte/go-consistent) анализирует указанные файлы и пакеты, подсчитывая количество использования той или иной альтернативы, предлагая заменить менее частые формы на идиоматичные в рамках анализируемой части проекта, те, у которых частота максимальна.

<spoiler title="Прямолинейный подсчёт"><hr>
На данный момент вес у каждого вхождения равен единице. Иногда это приводит к тому, что один файл диктует стиль всего пакета только из-за того, что в нём эта операция является более часто используемой. Особенно это ощутимо в отношении редких операций, вроде создания пустого `map`.

Насколько это оптимальная стратегия пока не понятно. Эту часть алгоритма будет несложно доработать или предоставить пользователям выбирать одну из нескольких предложенных.
<hr></spoiler>

Если `$(go env GOPATH)/bin` находится в системном `PATH`, то следующая команда установит `go-consistent`:

```bash
go get -v github.com/Quasilyte/go-consistent
go-consistent --help # Для проверки установки
```

Возвращаясь к границам консистентности, вот как проверить каждую из них:

- Проверить консистентность внутри одного файла можно с помощью запуска `go-consistent` на этом файле
- Консистентность внутри пакета вычисляется при запуске с одним аргументом-пакетом (или с указанием всех файлов этого пакета)
- Вычисление глобальной консистентности потребует передачи всех пакетов в качестве аргументов

`go-consistent` устроен таким образом, что может выдать ответ даже для огромных репозиториев, где загрузить все пакеты в память единовременно довольно сложно (по крайней мере, на персональной машине без огромного количества оперативной памяти).

Другая важная особенность - это zero configuration. Запуск `go-consistent` без каких-либо флагов и конфигурационных файлов - это то, что работает для 99% случаев.

> **Предупреждение**: первый запуск на проекте может выдать большое количество предупреждений. Это не значит, что код написан плохо, просто контролировать согласованность на таком микро уровне достаточно сложно и не стоит трудозатрат, если контроль выполняется исключительно вручную.

# [go-namecheck](https://github.com/Quasilyte/go-namecheck)

Снизить консистентность кода может непоследовательное именование параметров функции или локальных переменных.

Большинству Go программистов очевидно, что `erro` менее удачное имя для ошибки, чем `err`. А как насчёт `s` против `str`?

Задачу проверки консистентности имён переменных решить методами `go-consistent` нельзя. Здесь сложно обойтись без манифеста локальных конвенций.

[go-namecheck](https://github.com/Quasilyte/go-namecheck) определяет формат этого манифеста и позволяет проводить его валидацию, упрощая следование определённым в проекте нормам именования сущностей.

Например, можно указать, что для **параметров функций**, имеющих тип **string**, стоит использовать идентификатор `s` вместо `str`.

Выражается это правило следующий образом:

```js
{"string": {"param": {"str": "s"}}}
```

* `string` - регулярное выражение, которое захватывает интересующий тип
* `param` - область применимости правил замены (scope). Их может быть несколько
* Пара `"str": "s"` указывает на замену с `str` на `s`. Их может быть несколько

Вместо замены 1-к-1 можно использовать регулярное выражение, которое захватывает более чем один идентификатор. Вот, например, правило, которое требует замены префикса `re` у переменных типа `*regexp.Regexp` на суффикс `RE`. Другими словами, вместо `reFile` правило требовало бы использовать `fileRE`.

```js
{
  "regexp\\.Regexp": {
    "local+global": {"^re[A-Z]\\w*$": "use RE suffix instead of re prefix"}
  }
}
```

Все типы рассматриваются с игнорированием указателей. Любой уровень косвенности будет удалён, поэтому не нужно определять отдельные правила для указателей на тип и самого типа.

Файл, который описывал бы оба правила, выглядел бы так:

```js
{
  "string": {
    "param": {
      "str": "s",
      "strval": "s"
    },
  },
  "regexp\\.Regexp": {
    "local+global": {"^re[A-Z]\\w*$": "use RE suffix instead of re prefix"}
  }
}
```

Предполагается, что проект начинается с пустого файла. Затем, в определённый момент, на code review, происходит запрос на переименование переменной или поля в структуре. Естественной реакцией может быть просьба закрепить эти, ранее неформальные, требования, в виде верифицируемого правила в файле конвенций именования. В следующий раз проблему можно будет находить автоматически.

Устанавливается и используется `go-namecheck` аналогично `go-consistent`, за исключением лишь того, что для получения корректного результата вам не нужно запускать проверку на всём множестве пакетов и файлов.

# Заключение

Особенности, рассмотренные выше, не являются критичными по отдельности, но влияют на общую консистентность в совокупности. Мы рассмотрели однородность кода на микро уровне, который не зависит от архитектуры или других особенностей приложения, поскольку эти аспекты проще всего валидировать с почти нулевым количеством ложных срабатываний.

Если по описаниям выше [go-consistent](https://github.com/Quasilyte/go-consistent) или [go-namecheck](https://github.com/Quasilyte/go-namecheck) вам понравились, попробуйте запустить их на своих проектах. Обратная связь - по-настоящему ценный подарок для меня.

**Важно**: если у вас есть какая-то идея или дополнение, пожалуйста, скажите об этом!<br>Есть несколько способов:
* Напишите в комментариях к этой статье
* [Создайте issue go-consistent](https://github.com/Quasilyte/go-consistent/issues/new)
* Реализуйте свою утилиту и дайте миру о ней знать

> **Предупреждение**: добавлять `go-consistent` и/или `go-namecheck` в CI может быть чересчур радикальным действием. Запуск раз в месяц с последующей правкой всех несоответствий может быть более удачным решением.
