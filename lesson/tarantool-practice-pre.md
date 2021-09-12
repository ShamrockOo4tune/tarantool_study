---
title: Tarantool Practice
description:
---
# Tarantool

## Установка

```
sudo dnf install tarantool tarantool-devel
```

## Запуск

```
tarantool
```
- Работу с данными можно сразу производить внутри базы
- Для взаимодействия с базой используются скрипты на языке Lua

## Конфигурация базы данных

- `>`
  ```lua
  box.cfg{listen=3301}
  ```

## Создание спейса (таблицы)

- `>`
  ```lua
  box.schema.space.create('games')
  ```

## Схема спейса

- `>`
  ```lua
  box.space.games:format({
      {name="id", type="unsigned"}, 
      {name="name", type="string"},
      {name="category", type="string"},
      {name="rate", type="unsigned"}})
  ```

## Создание первичного ключа по полю id

- `>`
  ```lua
  box.space.games:create_index('pkey',
    {parts={'id'}})
  ```

## Данные

- `>`
  ```lua
  box.space.games:insert({1, "Deus Ex", "rpg", 10})
  box.space.games:insert({2, "Fallout", "rpg", 11})
  box.space.games:insert({3, "Witcher 3", "rpg", 9})
  box.space.games:insert({4, "Sims", "sim", 8})
  box.space.games:insert({5, "Minecraft", "craft", 10})
  box.space.games:insert({6, "NFS", "sim", 10})
  ```

## Запрос по первичному ключу

- Запрос одного конкретного тапла
- `>`
  ```lua
  box.space.games:select({1})
  ```
- Запрос начиная с конкретного тапла и в больше
- `>`
  ```lua
  box.space.games:select({1}, {iterator='GE'})
  ```
- Запрос начиная с конкретного тапла и в меньше
- `>`
  ```lua
  box.space.games:select({1}, {iterator='LE'})
  ```

## Вторичный индекс

- `>`
  ```lua
  box.space.games:create_index('category_rate',
    {
        parts={'category', 'rate'}
    })
  ```

## Запросы по вторичному

- Итерация по вторичному с указанным старшим полем
- `>`
  ```lua
  for _, t in box.space.games.index['category_rate']:pairs('rpg') do
    print(t)
  end
  ```
- Итерация по вторичному с указанными всеми полями
- `>`
  ```lua
  for _, t in box.space.games.index['category_rate']:pairs({'rpg', 9}) do
    print(t)
  end
  ```
- Итерация по вторичному с указанными всеми полями начиная от и далее 
- `>`
  ```lua
  for _, t in box.space.games.index['category_rate']:pairs({'rpg', 10}, {iterator='GE'}) do
    print(t)
  end
  ```

## Хранимые процедуры на lua

- Дадим права для `guest`
- `>`
  ```lua
  box.schema.user.grant('guest', 'super')
  ```
- Создадим функцию
- `>`
  ```lua
  function getdata(arg)
    if arg == nil then error("NO FULLSCAN") end
    local result = {}
    for _, t in box.space.games.index['category_rate']:pairs(arg, {iterator='GE'}) do
      table.insert(result, t)
    end
    return 'Data is', result
  end
  ```

### Python

- Установим коннектор
```
sudo pip3 install tarantool
```
- Подключение
```
python
```
- `>>>`
  ```python
  import tarantool
  c = tarantool.connect('127.0.0.1', 3301)
  c.call('getdata')
  ```
  - NO FULLSCAN
  ```python
  c.call('getdata', 'rpg')
  ```

## SQL

- tarantool
- `>`
  ```lua
  box.execute([[ SELECT * FROM "games"  ]])
  ```

- python
- `>>>`
  ```python
  c.call('box.execute', """  SELECT * FROM "games" """)
  ```