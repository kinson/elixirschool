%{
  version: "1.1.1",
  title: "Erlang Term Storage (ETS)",
  excerpt: """
  Erlang Term Storage (ETS) - это мощный инструмент для хранения данных. Он встроен в OTP и доступен для использования в Elixir.
В этом уроке мы рассмотрим: как с ним работать, и как он может быть задействован в наших приложениях.
  """
}
---

## Обзор

ETS - инструмент для хранения объектов Elixir и Erlang в памяти.
Он способен хранить огромные объемы данных и предоставляет доступ за фиксированное время.

Таблицы ETS создаются отдельными процессами и принадлежат им же.
Когда процесс-владелец завершает работу, его таблицы уничтожаются.
Вы можете иметь столько таблиц ETS, сколько хотите, единственным ограничением является память сервера. Предел может быть указан с помощью переменной среды `ERL_MAX_ETS_TABLES`.


## Создание таблиц

Таблицы создаются с помощью функции `new/2`, которая получает название таблицы и набор опций. Она возвращает идентификатор таблицы, который мы можем использовать в последующих операциях.

Для нашего примера мы создадим таблицу для хранения и получения пользователей по имени:

```elixir
iex> table = :ets.new(:user_lookup, [:set, :protected])
8212
```

К ETS таблицам, так же как и к GenServer, можно обращаться по имени вместо идентификатора.
Для этого нужно передать опцию `:named_table`.
Тогда получится обращаться к таблице напрямую по имени:

```elixir
iex> :ets.new(:user_lookup, [:set, :protected, :named_table])
:user_lookup
```

### Типы таблиц

В ETS доступны четыре типа таблиц:

+ `set` — Тип таблицы по умолчанию.
Одно значение на ключ.
Ключи всегда уникальны.
+ `ordered_set` — Похоже на `set`, но отсортировано по ключу.
Важно отметить, что сравнение ключей различается в пределах `ordered_set`.
Ключи не обязаны быть одинаковыми, важно чтобы они были сравнимы.
Например, 1 и 1.0 считаются равными.
+ `bag` — Много объектов на ключ, но записи в целом уникальны.
+ `duplicate_bag` — Много объектов на ключ, возможны дубликаты.

### Права доступа

Права доступа в ETS похожи на те, что существуют внутри модулей:

+ `public` — И чтение, и запись доступны всем процессам.
+ `protected` — Чтение доступно для всех процессов.
Запись - только владельцу.
Это тип по умолчанию.
+ `private` — Любой доступ доступен только процессу-владельцу.

## Состояние гонки

Состояние гонки возможно, если более чем один процесс может записывать в таблицу - будь то с помощью `:public` или с помощью сообщений.
Например, два процесса читают значение счетчика `0`, увеличивают его и печатают `1`; в результате отразится только один инкремент.

В частности, для счетчиков [:ets.update_counter/3](http://erlang.org/doc/man/ets.html#update_counter-3) обеспечивает атомарное обновление и чтение.
В других случаях может потребоваться чтобы процесс владельца выполнял пользовательские атомарные операции в ответ на сообщения, такие как "добавить это значение в список по ключу `:results`".

## Вставка данных

В ETS нет определенной структуры данных.
Единственным ограничением является то, что данные должны храниться как кортеж, в котором первым элементом является ключ.
Для добавления новой записи можно использовать `insert/2`:

```elixir
iex> :ets.insert(:user_lookup, {"doomspork", "Sean", ["Elixir", "Ruby", "Java"]})
true
```

Когда мы используем `insert/2` вместе с типом таблиц `set` или `ordered_set`, существующие данные будут заменены.
Для избежания этого есть метод `insert_new/2`, который всегда возвращает `false` для существующих ключей:

```elixir
iex> :ets.insert_new(:user_lookup, {"doomspork", "Sean", ["Elixir", "Ruby", "Java"]})
false
iex> :ets.insert_new(:user_lookup, {"3100", "", ["Elixir", "Ruby", "JavaScript"]})
true
```

## Получение данных

ETS предоставляет нам несколько методов для получения данных.
Мы рассмотрим, как получить данные по ключу, а также через различные формы сопоставления с образцом.

Самым эффективным и идеальным, является получение по ключу.
В то время, как сопоставление очень удобно для поиска, это долго и потому должно применяться с осторожностью на больших объёмах данных.

### Поиск по ключу

Имея ключ, мы можем использовать `lookup/2` для получения всех записей, у которых есть этот ключ:

```elixir
iex> :ets.lookup(:user_lookup, "doomspork")
[{"doomspork", "Sean", ["Elixir", "Ruby", "Java"]}]
```

### Простой поиск

ETS изначально создавался для Erlang, потому синтаксис может выглядеть _слегка_ необычно.

Для указания переменной в нашем сопоставлении мы должны использовать атомы `:"$1"`, `:"$2"`, `:"$3"` и так далее.
Номер переменной отображает порядок в результате.
Для переменных, которые нам не интересны, можно использовать `:_`.

Значения также могут быть использованы для поиска, но только переменные будут возвращены как результат.
Давайте попробуем воспользоваться этими знаниями:

```elixir
iex> :ets.match(:user_lookup, {:"$1", "Sean", :_})
[["doomspork"]]
```

Давайте рассмотрим другой пример, как переменные влияют на порядок вывода результатов:

```elixir
iex> :ets.match(:user_lookup, {:"$99", :"$1", :"$3"})
[["Sean", ["Elixir", "Ruby", "Java"], "doomspork"],
 ["", ["Elixir", "Ruby", "JavaScript"], "3100"]]
```

А что, если мы хотим получить не список, а оригинальный объект целиком? Можно вызвать `match_object/2`, который вне зависимости от переменных вернет объект целиком:

```elixir
iex> :ets.match_object(:user_lookup, {:"$1", :_, :"$3"})
[{"doomspork", "Sean", ["Elixir", "Ruby", "Java"]},
 {"3100", "", ["Elixir", "Ruby", "JavaScript"]}]

iex> :ets.match_object(:user_lookup, {:_, "Sean", :_})
[{"doomspork", "Sean", ["Elixir", "Ruby", "Java"]}]
```

### Продвинутый поиск

Мы уже изучили, как работать с простыми случаями, но что, если нужно сделать что-то похожее на SQL запрос? К счастью, у ETS есть более гибкий синтаксис.
Для поиска данных с использованием `select/2` нам нужно составить список кортежей, в каждом из которых будет по три элемента.
Эти кортежи отображают наш запрос и формат вывода.

Переменные сопоставления и две новых переменных, `:"$$"` и `:"$_"`, могут быть использованы для формирования возвращаемого значения.
`:"$$"` получает результат как список, а `:"$_"` получает оригинальные объекты данных.

Давайте возьмем один из предыдущих примеров с `match/2` и превратим его в `select/2`:

```elixir
iex> :ets.match_object(:user_lookup, {:"$1", :_, :"$3"})
[{"doomspork", "Sean", ["Elixir", "Ruby", "Java"]},
 {"3100", "", ["Elixir", "Ruby", "JavaScript"]}]

{% raw %}iex> :ets.select(:user_lookup, [{{:"$1", :_, :"$3"}, [], [:"$_"]}]){% endraw %}
[{"doomspork", "Sean", ["Elixir", "Ruby", "Java"]},
 {"3100", "", ["Elixir", "Ruby", "JavaScript"]}]
```

Несмотря на то, что `select/2` позволяет лучше управлять получением записей, синтаксис неудобен.
Для того чтобы решить эту проблему, в ETS есть функция `fun2ms/1`, которая превращает функции в match_spec.
С этим инструментом мы можем создавать запросы, используя уже известный синтаксис функций.

Давайте воспользуемся `fun2ms/1` и `select/2` для получения всех имён пользователей, знающих более двух языков:

```elixir
iex> fun = :ets.fun2ms(fn {username, _, langs} when length(langs) > 2 -> username end)
{% raw %}[{{:"$1", :_, :"$2"}, [{:>, {:length, :"$2"}, 2}], [:"$1"]}]{% endraw %}

iex> :ets.select(:user_lookup, fun)
["doomspork", "3100"]
```

Хотите узнать больше о настройках поиска? Официальная документация Erlang по match_spec доступна по [этой](http://www.erlang.org/doc/apps/erts/match_spec.html) ссылке.

## Удаление данных

### Удаление записей

Удаление строк так же просто, как и `insert/2` или `lookup/2`.
С функцией `delete/2` нам нужна только таблица и ключ.
Этот вызов удаляет и ключ, и значения в нем.

```elixir
iex> :ets.delete(:user_lookup, "doomspork")
true
```

### Удаление таблиц

Таблицы ETS собираются сборщиком мусора только, если процесс-родитель закончил работу.
Иногда может быть необходимо удалить целую таблицу, не уничтожая сам процесс.
Для этого мы можем использовать `delete/1`:

```elixir
iex> :ets.delete(:user_lookup)
true
```

## Пример использования ETS

Давайте попробуем создать простой кэш для затратных операций, используя всё, что мы рассмотрели выше.
Мы реализуем функцию `get/4` для получения модуля, функции, аргументов и опций.
На данный момент единственной опцией, которая нас интересует, будет `:ttl`.

Для этого примера мы будем считать, что таблица ETS была создана как часть другого процесса, например, супервизора:

```elixir
defmodule SimpleCache do
  @moduledoc """
  Простой кэш, использующий ETS.
  """

  @doc """
  Получить закэшированное значение или вызвать переданную функцию и запомнить результат.
  """
  def get(mod, fun, args, opts \\ []) do
    case lookup(mod, fun, args) do
      nil ->
        ttl = Keyword.get(opts, :ttl, 3600)
        cache_apply(mod, fun, args, ttl)

      result ->
        result
    end
  end

  @doc """
  Найти закэшированный результат и проверить его давность.
  """
  defp lookup(mod, fun, args) do
    case :ets.lookup(:simple_cache, [mod, fun, args]) do
      [result | _] -> check_freshness(result)
      [] -> nil
    end
  end

  @doc """
  Сравнить результат выполнения с текущим системным временем.
  """
  defp check_freshness({mfa, result, expiration}) do
    cond do
      expiration > :os.system_time(:seconds) -> result
      :else -> nil
    end
  end

  @doc """
  Вызвать функцию, вычислить время жизни и закэшировать результат.
  """
  defp cache_apply(mod, fun, args, ttl) do
    result = apply(mod, fun, args)
    expiration = :os.system_time(:seconds) + ttl
    :ets.insert(:simple_cache, {[mod, fun, args], result, expiration})
    result
  end
end
```

Для демонстрации кэша мы будем использовать функцию, которая возвращает системное время с временем жизни кэша в 10 секунд.
Как мы увидим в примере ниже, закэшированный результат возвращается до того момента, как истечет срок жизни:

```elixir
defmodule ExampleApp do
  def test do
    :os.system_time(:seconds)
  end
end

iex> :ets.new(:simple_cache, [:named_table])
:simple_cache
iex> ExampleApp.test
1451089115
iex> SimpleCache.get(ExampleApp, :test, [], ttl: 10)
1451089119
iex> ExampleApp.test
1451089123
iex> ExampleApp.test
1451089127
iex> SimpleCache.get(ExampleApp, :test, [], ttl: 10)
1451089119
```

Если попробовать еще раз после 10 секунд, будет возвращен новый результат:

```elixir
iex> ExampleApp.test
1451089131
iex> SimpleCache.get(ExampleApp, :test, [], ttl: 10)
1451089134
```

Как видно выше, мы смогли имплементировать масштабируемый и быстрый сервер кэширования без внешних зависимостей - и это только один из вариантов использования ETS!

## Хранение данных на диске в ETS

Как мы теперь знаем, ETS использует хранилище в памяти. Но что если нам нужно хранение данных на диске? Для этого у нас есть DETS (Disk Based Term Storage).
API для ETS и DETS - взаимозаменяемы за исключением того, как создается таблица.
Для DETS используется `open_file/2` и для него не требуется параметр `:named_table`:

```elixir
iex> {:ok, table} = :dets.open_file(:disk_storage, [type: :set])
{:ok, :disk_storage}
iex> :dets.insert_new(table, {"doomspork", "Sean", ["Elixir", "Ruby", "Java"]})
true
iex> select_all = :ets.fun2ms(&(&1))
[{:"$1", [], [:"$1"]}]
iex> :dets.select(table, select_all)
[{"doomspork", "Sean", ["Elixir", "Ruby", "Java"]}]
```

Если выйти из `iex` и посмотреть в текущую папку, то там будет новый файл `disk_storage`:

```shell
$ ls | grep -c disk_storage
1
```

Последнее, о чем хотелось бы написать, это то, что DETS не поддерживает `ordered_set`. Только `set`, `bag`, и `duplicate_bag`.
