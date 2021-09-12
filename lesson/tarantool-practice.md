---
title: Tarantool Practice
description:
---
# Tarantool

## Приложение

Сделаем in-memory хранилище и web приложение, которое будет показывать 
отзывы туристов о тех или иных достопримечательностях.

## ТЗ

- html/js интерфейса с картой
  - Отображение текущих отзывов
  - Отображение диалога для создания отзыва
- Бэкенд и in-memory хранилище данных на Tarantool 
  - Индексация
- В Tarantool комментарии туристов будут храниться в таблице `places` в следующей структуре
    - Она похожа на плоский GeoJSON
    ```yaml
    _id: string
    type: string
    geometry.type: string
    geometry.coordinates: array of double
    properties.comment: string
    ```

## Зависимости

- Установить Tarantool

```
sudo dnf install tarantool tarantool-devel
```

## Код

- Создадим директорию
    ``` bash
    mkdir thestreets
    cd thestreets
    ```
- Установим встроенный фреймворк для приложений cartridge
    ```bash
    tarantoolctl rocks install http
    ```

### Приложение

- Создадим lua модуль, в котором реализуем:
  - Обработку http запросов
  - Хранение и индексацию отзывов
- `streets.lua`
    ```lua
    local log = require('log')
    local fio = require('fio')
    local uuid = require('uuid')
    local json = require('json')

    --[[
        Отдаём фронтенд часть браузеру
    ]]
    local function streets(request)
        return { body=fio.open('templates/index.html'):read() }
    end

    --[[
        Создаём новый отзыв и сохраняем в таблицу
    ]]
    local function newplace(request)
        local place = {}
        -- json от фронтенда
        local obj = request:json()

        --[[ Генерируем уникальный идентификатор для отзыва]]
        place['_id'] = uuid.str()
        obj['_id'] = place['_id']
        --[[ Делаем структуру объекта плоской ]]
        place['type'] = obj['type']
        place['geometry.type'] = obj['geometry']['type']
        place['geometry.coordinates'] = obj['geometry']['coordinates']
        place['properties.comment'] = obj['properties']['comment']

        --[[
            Создаём сущность для таблицы
        ]]
        local t, err = box.space.streets:frommap(place)
        if err ~= nil then
            log.error(tostring(err))
            return {code=503, body=tostring(err)}
        end
        --[[
            Вставляет объект
        ]]
        box.space.streets:insert(t)

        return { body=json.encode(obj) }
    end

    --[[
        Утилита для вычисления расстояния
        Потребуется чуть ниже
    ]]
    local function distance(x, y, x2, y2)
        return math.sqrt(math.pow(x2-x, 2) + math.pow(y2-y, 2))
    end

    --[[
        Функция возвращает объекты на карте
        ближайшие к указанной точке
    ]]
    local function places(request)
        local result = {}

        local limit = 1000
        local x = tonumber(request:param('x'))
        local y = tonumber(request:param('y'))
        local dist = tonumber(request:param('distance'))

        x = x or 1
        y = y or 1
        dist = dist or 0.2

        --[[
            Итерируемся по таблице начиная с ближайщих к указанной точке объектов
        ]]
        for _, place in box.space.streets.index.spatial:pairs({x, y}, {iterator='NEIGHBOR'}) do
            -- Если объект уже очень далеко, прерываем итерацию
            if distance(x, y, place['geometry.coordinates'][1], place['geometry.coordinates'][2]) > dist then
                break
            elseif place['properties.rate']<4 then
                print ("id", place['id'], ", rate", place['properties.rate', " ;"])
                break
            end

            -- Создаём GeoJSON
            local obj = {
                ['_id'] = place['_id'],
                type = place['type'],
                geometry = {
                    type = place['geometry.type'],
                    coordinates = place['geometry.coordinates'],
                },
                properties = {
                    comment = place['properties.comment'],
                    rate = place['properties.rate'],
                },
            }
            table.insert(result, obj)
            limit = limit - 1
            if limit == 0 then
                break
            end
        end
        return {code=200,
                body=json.encode(result)}
    end

    --[[
        Инициализации
    ]]

    box.cfg{}

    --[[
        Создаём таблицу для хранения отзывов на карте
    ]]
    box.schema.space.create('streets', {if_not_exists=true})
    box.space.streets:format({
            {name="_id", type="string"},
            {name="type", type="string"},
            {name="geometry.type", type="string"},
            {name="geometry.coordinates", type="array"},
            {name="properties.comment", type="string"},
    })
    --[[ Создаём первичный индекс ]]
    box.space.streets:create_index('primary', {
                                    parts={{field="_id", type="string"}},
                                    type = 'TREE',
                                    if_not_exists=true,})
    --[[ Создаём индекс для координат ]]
    box.space.streets:create_index('spatial', {
                                    parts = {{ field="geometry.coordinates", type='array'} },
                                    type = 'RTREE', unique = false,
                                    if_not_exists=true,})

    --[[ Настраиваем http сервис ]]
    local httpd = require('http.server').new('0.0.0.0', 8081)
    local router = require('http.router').new()
    httpd:set_router(router)
    router:route({path="/"}, streets)
    router:route({path="/newplace"}, newplace)
    router:route({path="/places"}, places)

    httpd:start()
    ```

### Фронтенд

- Фронтенд страница с картой и кнопками для созданию отзывов
- Фронтенд преобразует коориднаты на шаре в координаты на плоскости, 
  чтобы Tarantool индекс работал как надо
- `templates/index.html`
    ```html
    <html lang="en">

    <head>
        <title>The Map</title>
        <link rel="stylesheet" href="https://unpkg.com/leaflet@1.7.1/dist/leaflet.css" crossorigin="" />
        <script src="https://unpkg.com/leaflet@1.7.1/dist/leaflet.js" crossorigin=""></script>
        <script src="https://unpkg.com/leaflet-providers@1.0.13/leaflet-providers.js" crossorigin=""></script>
    </head>

    <body>
        <!-- Отображение карты и текущих координат -->
        <div id="mapid" style="height:600px"></div>
        <div id="centerid">Center:</div>
        <span>Show from rate: </span><input id="rate" name="rate" value="1" onKeyPress='ratechange();'></input>
        <script>
            /*
            * Виджет карты
            */
            var mymap = L.map('mapid',
                { 'tap': false })
                .setView([59.95184617254149, 30.30683755874634], 13);

            /*
            * Слой карты с домами, улицами и т.п.
            */
            var OpenStreetMap_Mapnik = L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                maxZoom: 19,
                attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
            }).addTo(mymap);

            /*
            * Слой карты с комментариями
            * Этот слой будут наполнять пользователи
            */
            var got = {};
            var layer = L.geoJSON(null, {
                "onEachFeature": function (feature, layer) {
                    got[feature['_id']] = true;
                    if (feature.properties && feature.properties.comment) {
                        var description = feature.properties.comment;
                        if (feature.properties.rate) {
                            description = description + '; rate ' + feature.properties.rate
                        }
                        var popup = layer.bindPopup(description);
                    }
                },
                "filter": function (feature) {
                    /* Филтьруем элементы, которые уже были отображены на карте */
                    return got[feature['_id']] != true;
                }
            }).addTo(mymap);

            /*
            * Диалог для комметария
            */
            function onMapClick(e) {
                var response = window.prompt('What do you think about this place?', 'not bad');
                if (response != null) {
                    var rate = window.prompt('Rate place from 1 (worst) to 5 (best)', '5');
                    if (rate == null)
                        rate = 5

                    var p = mymap.project(e.latlng, 1);

                    var data = {
                        "type": "Feature",
                        "geometry": {
                            "type": "Point",
                            "coordinates": [p.x, p.y]
                        },
                        "properties": {
                            "comment": response,
                            "rate": Number(rate)
                        }
                    }

                    fetch("/newplace", {
                        method: "POST",
                        body: JSON.stringify(data)
                    }).then(function (res) {
                        res.json().then(function (data) {
                            var l = mymap.unproject(L.point(data['geometry']['coordinates'][0], data['geometry']['coordinates'][1]), 1);
                            data['geometry']['coordinates'][0] = l.lng;
                            data['geometry']['coordinates'][1] = l.lat;

                            layer.addData(data);
                        })
                    });
                }
            }

            mymap.on('click', onMapClick);

            /*
            * Загрузка комметариев с сервера, расположенных поблизости от текущих
            * координат
            */
            function getPlaces() {
                var request = {};
                var p = mymap.project(mymap.getCenter(), 1);
                var options = {
                    "x": p.x,
                    "y": p.y,
                };

                options['rate'] = document.getElementById("rate").value;

                fetch("/places?" + new URLSearchParams(options))
                    .then(function (res) {
                        res.json().then(function (dataarray) {
                            dataarray.forEach(function(data) {
                                var l = mymap.unproject(L.point(data['geometry']['coordinates'][0], data['geometry']['coordinates'][1]), 1);
                                data['geometry']['coordinates'][0] = l.lng;
                                data['geometry']['coordinates'][1] = l.lat;

                                layer.addData(data); 
                            });
                        })
                    });
            }

            /*
            * Загружаем комметарии при навигации по карте
            */
            var timerId = null
            function onMapMove(e) {
                document.getElementById("centerid").innerHTML = "Center: " + mymap.getCenter();

                if (timerId == null) {
                    timerId = setTimeout(function () {
                        getPlaces();
                        timerId = null
                    }, 1000);
                }
            };

            mymap.on('move', onMapMove);

            document.getElementById("centerid").innerHTML = "Center: " + mymap.getCenter();
            timerId = setTimeout(function () {
                getPlaces();
                timerId = null
            }, 1000);

            function ratechange() {
                got = {};
                layer.clearLayers();

                timerId = setTimeout(function () {
                    getPlaces();
                    timerId = null
                }, 1000);
            }
        </script>
    </body>

    </html>
    ```

## Запуск 

```
tarantool streets.lua
```

## Самостоятельно

- Добавить бальную оценку пользователя от 1 до 5
- Настроить фильтрацию для отображения только тех комментариев,
  которые выше определённого балла
- Реализацию можно выполнить только на уровне Tarantool бэкенда

**Рекомендации**
- Добавить поле rate в объекты
- Добавить условие для фильтрации по полю rate в итерации в запросе данных