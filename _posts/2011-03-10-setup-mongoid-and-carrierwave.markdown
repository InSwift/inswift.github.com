---
title: Подружить Mongoid и CarrierWave
layout: post
tags:
  - ruby on rails
  - mongoid
  - carrierwave
---

Я пишу одно приложение на рельсах с использованием [MongoDB](http://mongodb.org).
Для ruby есть [несколько](http://ruby-toolbox.com/categories/mongodb_clients.html)
библиотек для работы с этой СУБД. Я решил использовать
[Mongoid](https://github.com/mongoid/mongoid) как наиболее популярный в списке.

Не так много плагинов работают с Mongoid, и это недостаток. Например работа с
[Formtastic](https://github.com/justinfrench/formtastic) немного
[неудобна](https://github.com/bowsersenior/formtastic_with_mongoid_tutorial).
Ситуацию усложняет ещё и то, что в отличие от `ActiveRecord` в `Mongoid`
есть не только ссылки между моделями
(`referenced_in`, `references_many`, аналог `belongs_to` и `has_many`), но и
встраивание (`embedded_in`, `embeds_many`). Некоторые плагины некорректно работают
с такими встроенными моделями.

[CarrierWave](https://github.com/jnicklas/carrierwave) -- один из немногих что работают
на ура. [Здесь](http://socialmemorycomplex.net/2010/06/02/gridfs-with-mongoid-and-carrierwave-on-rails-3/)
можно прочесть о настройке. Заработало почти сразу, только в одна строчка в конфиге бросала
исключение. 

<script src="https://gist.github.com/973651.js?file=gistfile1.rb"></script>

следует поменять на:

<script src="https://gist.github.com/973651.js?file=gistfile2.rb"></script>

Кроме того мне понадобилось прикреплять несколько файлов к одной модели.
Ничего похожего на `mount_uploaders`
(оно и понятно: как быть с реляционными базами данных? В массиве хранить?)
Поэтому мне пришлось сделать дополнительную модель `Image`, для хранения
одного файла. Далее я сделал 1:N отношение, и получил несколько файлов для
одной модели.

<script src="https://gist.github.com/973651.js?file=gistfile3.rb"></script>

<script src="https://gist.github.com/973651.js?file=gistfile4.rb"></script>

При работе с формами почему-то не работал корректно `fields_for` для `images`.
Не желал распознавать как массив. Нагуглил решение `accepts_nested_attributes_for :images`.

Форма выглядит примерно так:
<script src="https://gist.github.com/973651.js?file=gistfile5.erb"></script>

Чтобы было для чего создавать поля предварительно заполняем `images`
пустыми моделями. Метод `new` выглядит что-то вроде:

<script src="https://gist.github.com/973651.js?file=gistfile6.rb"></script>

И здесь снова неприятность. По какой-то причине метод `save` не сохраняет
вложенные модели. Поэтому метод `create` выглядит примерно так:

<script src="https://gist.github.com/973651.js?file=gistfile7.rb"></script>

Теперь оно наконец работает.
