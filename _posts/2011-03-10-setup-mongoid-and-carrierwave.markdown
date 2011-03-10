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

{% highlight ruby %}
config.grid_fs_host = Mongoid.config.master.connection.host
{% endhighlight %}
следует поменять на:
{% highlight ruby %}
config.grid_fs_host = Mongoid.database.connection.primary_pool.host
{% endhighlight %}

Кроме того мне понадобилось прикреплять несколько файлов к одной модели.
Ничего похожего на `mount_uploaders`
(оно и понятно: как быть с реляционными базами данных? В массиве хранить?)
Поэтому мне пришлось сделать дополнительную модель `Image`, для хранения
одного файла. Далее я сделал 1:N отношение, и получил несколько файлов для
одной модели.

{% highlight ruby%}
class Image
  include Mongoid::Document

  embedded_in :market_item

  mount_uploader :attach, ImageUploader
end
{% endhighlight %}

{% highlight ruby %}
class MarketItem
  include Mongoid::Document

  embeds_many :images, :stored_as => :array

  # BUGFIX
  # next code-line needed for fix bug array described here:
  # http://tinyurl.com/6yntj3r 
  accepts_nested_attributes_for :images
end
{% endhighlight %}

При работе с формами почему-то не работал корректно `fields_for` для `images`.
Не желал распознавать как массив. Нагуглил решение `accepts_nested_attributes_for :images`.

Форма выглядит примерно так:
{% highlight erb %}
<%= form_for @market_item, :html => {:multipart => true} do |f| -%>
  <%= f.fields_for :images do |builder| -%>
    <%= builder.label :attach %>
    <%= builder.file_field 'attach' %>
  <% end -%>

  <%= f.submit "Отправить" %>
<% end -%>
{% endhighlight %}

Чтобы было для чего создавать поля предварительно заполняем `images`
пустыми моделями. Метод `new` выглядит что-то вроде:

{% highlight ruby %}
# GET /market_items/new
# GET /market_items/new.xml
def new
  @market_item = MarketItem.new
  3.times { @market_item.images.build }

  respond_to do |format|
    format.html # new.html.erb
    format.xml  { render :xml => @market_item }
  end
end
{% endhighlight %}

И здесь снова неприятность. По какой-то причине метод `save` не сохраняет
вложенные модели. Поэтому метод `create` выглядит примерно так:

{% highlight ruby %}
# POST /market_items
# POST /market_items.xml
def create
  @market_item = MarketItem.new(params[:market_item])
  
  respond_to do |format|
    if @market_item.save

      # BUGFIX
      # Don't know why, but nested images does not saves automaticly
      # bug a bit described here (at end):
      # http://flux88.com/blog/using-carrierwave-with-mongoid/
      @market_item.images.each {|x| x.save!}

      format.html { redirect_to(@market_item,
                                :notice => 'created.') }
      format.xml  { render :xml => @market_item,
                           :status => :created,
                           :location => @market_item }
    else
      format.html { render :action => "new" }
      format.xml  { render :xml => @market_item.errors,
                           :status => :unprocessable_entity }
    end
  end
end
{% endhighlight %}


Теперь оно наконец работает.
