---
layout: post
title: Обезьяньи заплатки по человечьи. Часть 1.
tags:
  - monkey patching
  - oop
  - ruby
---

Ведь правда, классная всё же это вещь, [monkey patching](http://en.wikipedia.org/wiki/Monkey_patch).

Суть проста. У нас есть какой-то объект, который имеет определённый набор методов.
В любой момент во время выполнения мы можем добавить
новые методы, или переопределить старые.

Звучит круто, правда? Да...

Именно из-за этой фичи в Rails можно писать что-нибудь вроде:

<script src="https://gist.github.com/973566.js?file=gistfile1.rb"></script>

Посмотрим как это устроено. Для ковыряния во внутренностях руби,
удобно использовать [Rubinius](http://http://rubini.us), т.к. он
предоставляет очень много информации о среде, да и гораздо легче
читать код на руби, чем на Си, когда необходимо понять как работает тот
или иной builtin метод.

# Внутри ActiveSupport

<script src="https://gist.github.com/973566.js?file=gistfile2.txt"></script>

Как можно увидеть, нужный код лежит в геме activesupport в `lib/active_support/core_ext/numeric/time.rb`.

[Здесь](https://github.com/rails/rails/blob/b2d94322e6f2c2324154465147938ca8b16c610d/activesupport/lib/active_support/core_ext/numeric/time.rb#L52) можно увидеть описание метода `weeks` и прочих подобных ему.
Данные методы определяются для класса `Numeric`.
Зачем нужен этот класс догадаться достаточно легко.

<script src="https://gist.github.com/973566.js?file=gistfile4.txt"></script>

В этом классе определяется набор методов общий для всех классов
описывающих числа. Мне было не совсем ясно какие могут быть общие методы
у числовых классов. Впрочем список методов можно легко получить:

<script src="https://gist.github.com/973566.js?file=gistfile5.txt"></script>

И к такому набору методов добавляются заветные `days` и `weeks`. Что ж, примерно понятно. 

Вот так просто можно добавить любые методы к классу.

Рассмотрим собственно метод `weeks`. Выглядит он так:

<script src="https://gist.github.com/973566.js?file=gistfile3.rb"></script>

Для того чтобы понять что же всё-таки происходит необходимо посмотреть что есть `ActiveSupport::Duration`,
что собственно можно сделать [здесь](https://github.com/rails/rails/blob/186e3c71f95316b94e728eb62b519d074d27cea3/activesupport/lib/active_support/duration.rb).

Всё примерно понятно. Неясно только как получается так:

<script src="https://gist.github.com/973566.js?file=gistfile6.txt"></script>

Откуда `Fixnum`? Ответ кроется в определении метода `method_missing` (строка [101](https://github.com/rails/rails/blob/186e3c71f95316b94e728eb62b519d074d27cea3/activesupport/lib/active_support/duration.rb#L101)),
который перенаправляет все вызовы объекту который хранится в `@value`, т.е.
нашему значению.

Погодите, но ведь `method_missing` обрабатывает только вызовы методов, которые
не определены для этого объекта. Как-так получилось что метод `class` не определён?
Дело в том, что `ActiveSupport::Duration` неспроста наследуется от `BasicObject`.
В этом объекте определён только минимальный набор методов, и метода `class` там нет =).
