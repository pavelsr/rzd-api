# Api сайта rzd.ru

[![Latest Stable Version](https://poser.pugx.org/visavi/rzd-api/v/stable)](https://packagist.org/packages/visavi/rzd-api)
[![Total Downloads](https://poser.pugx.org/visavi/rzd-api/downloads)](https://packagist.org/packages/visavi/rzd-api)
[![Latest Unstable Version](https://poser.pugx.org/visavi/rzd-api/v/unstable)](https://packagist.org/packages/visavi/rzd-api)
[![License](https://poser.pugx.org/visavi/rzd-api/license)](https://packagist.org/packages/visavi/rzd-api)

[Описание установки](https://github.com/visavi/rzd-api/blob/master/docs/install.md)

[Описание интерфейса пользователя](https://github.com/visavi/rzd-api/blob/master/docs/auth.md)

### Что умеет Api 
* Получает маршруты в одну точку
* Получает маршруты туда-обратно
* Получает список вагонов выбранного поезда
* Получает список станций в пути следования выбранного маршрута
* Получает список кодов станций (Поиск по первым символам города)

### Пример запроса
```php
<?php

$config = new Rzd\Config();

// Set language
$config->setLanguage('en');

// Set userAgent
$config->setUserAgent('Mozilla/5.0 (iPhone; CPU iPhone OS 12_1_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.0 Mobile/15E148 Safari/604.1');

// Set referer
$config->setReferer('https://ticket.rzd.ru/');

// Enable debug mode
$config->setDebugMode(true);

// Set proxy
$config->setProxy('https://username:password@192.168.16.1:10');

// Set timeout
$config->setTimeout(10);

//$config не обязателен
$api = new Rzd\Api($config);

// В примере выполняется поиск маршрута САНКТ-ПЕТЕРБУРГ - МОСКВА (только с билетами) на завтра
$params = [
    'dir'          => 0, // 0 - только в один конец, 1 - туда-обратно
    'tfl'          => 3, // 3 - поезда и электрички, 2 - электрички, 1 - поезда 
    'checkSeats'   => 1, // 1 - только с билетами, 0 - все поезда
    //'withoutSeats' => 'y', // Если checkSeats = 0, то этот параметр тоже необходим
    // Коды станций можно получить отдельным запросом
    'code0'        => '2004000', // код станции отправления
    'code1'        => '2000000', // код станции прибытия
    'dt0'          => 'дата на завтра d.m.Y',
    'md'           => 0, // 0 - без пересадок, 1 - с пересадками
];

$routes = $api->trainRoutes($params);
```

### Процесс приобретения билетов на сайте pass.rzd.ru разделен на несколько этапов

#### Открытая часть
Выбор маршрута - выбор поезда - выбор вагона

#### Закрытая часть
* Информация о пассажирах - Проверка заказа - Оплата заказа - Подтверждение заказа

### Этапы
* В первом этапе пользователь указывают станцию отправления и станцию прибытия поезда, а также дату желаемой поездки.
В этот момент на сайте pass.rzd.ru происходит отправка ajax-запроса, с которым мы и будем работать, запрос возвращает сформированный JSON пакет с ответом, в нем и находится требуемая нами информация или сообщение об ошибке

* Во втором этапе мы можем выбрать необходимый нам поезд и получить полную информацию о свободных местах

* В третьем этапе необходимо выбрать места и заполнить данные необходимые на оплаты и регистрации на сайте

Допустимые запросы через Curl (POST и GET)
Для обхода защиты сайта необходимо предварительно отправить запрос для получения cookies и номера идентификатора RID (REQUEST_ID)
Вторым запросом подставляем уникальный идентификатор RID и отправляем cookie

### Ответы с сайта
Статус ответа содержится в переменной result
RID - означает что сайт выдал нам уникальный идентификатор и куки
OK - получен полный ответ с запрошенными нами данными
Во всех остальных ответах Error или FAIL означает ошибку получения данных

### Получение cookie
Каждый запрос к сайту должен содержать куки примерного вида:
* lang=ru - текущий язык
* JSESSIONID=0000w74wcMhGMfeoE6ibmsh4i4W:17obq9kpt - уникальный ключ
* AuthFlag=false - авторизован ли пользователь на сайте

## Пример запроса

Все запросы идут на адрес http://pass.rzd.ru/timetable/public/ru?layer_id=подкатегория&ключ=значение

Где подкатегория это
* 5827 - выбор маршрута (Получения списка поездов)
* 5764 - детальная информация выбранному по поезду, список вагонов
* 5804 - просмотр маршрута со всеми остановками (Вроде больше не работает, реализовано по-другому)

### Первый запрос
https://pass.rzd.ru/timetable/public/ru?layer_id=5827&dir=0&tfl=3&checkSeats=1&code0={{code_from}}&dt0={{date}}&code1={{code_to}}&dt1={{date}}

### Второй и следующие запросы
https://pass.rzd.ru/timetable/public/ru?layer_id=5827&rid={{rid}}

Второй запрос выполняется с уже полученным нами уникальным идентификатором который хранит в себе данные предыдущего запроса и куками
Поэтому в целях оптимизации можно не отправлять некоторые параметры указанные нами в первом запросе

## Реализованные запросы

Необходимо реализовать отдачу данных через ajax-запросы, в текущих примерах это не реализовано

### trainRoutes - получает маршруты поездов, количество свободных мест, цены итд в нем в один конец

![Маршруты](https://github.com/visavi/rzd-api/blob/master/screens/trainRoute.png)

Принимает параметры
обязательные параметр при первом запросе
* layer_id - подкатегория (5827)

Необязательные параметр при повторном запросе
* dir - 0 - только в один конец, 1 - туда-обратно
* tfl - 3 - поезда и электрички, 2 - электрички, 1 - поезда 
* checkSeats - 0, 1 - поиск в поездах только если есть свободные места
* code0 - код станции отправления
* code1 - код станции прибытия
* dt0 - дата отправления
* md - маршруты с пересадками (1 - с пересадками, 0 - только прямые рейсы)

Возвращает массив поездов и свободных мест
* from - название станции отправления (САНКТ-ПЕТЕРБУРГ)
* where - название станции прибытия (КИРОВ ПАСС)
* date - дата отправления (27.03.2016)
* fromCode - код станции отправления (2004000)
* whereCode - код станции прибытия (2060600)

Массив поездов содержит
* date0 - дата отправления
* date1 - дата прибытия
* time0 - время отправления
* time1 - время прибытия
* route0 - код станции отправления С-ПЕТ-ЛАД
* route1 - код станции прибытия ТЮМЕНЬ
* number - номер поезда
* timeInWay - время в пути
* brand - Название поезда (Демидовский экспресс)
* carrier - тип поезда ФПК (Фирменный)

* cars - массив свободных мест купе, плацкарт и люкс
* cars.freeSeats - кол.  свободных мест
* cars.itype
* cars.servCls
* cars.tariff - стоимость билета
* cars.pt - баллы
* cars.typeLoc - полное наименование (Плацкартный, СВ, Купе, Люкс)
* cars.type - сокращенное наименование (Купе, плац, люкс)

### trainRoutesReturn - получает маршруты поездов, количество свободных мест, цены итд, туда-обратно
![Поезда](https://github.com/visavi/rzd-api/blob/master/screens/trainRouteReturn.png)

Принимает параметры
обязательные параметр при первом запросе
* layer_id - подкатегория (5827)

Необязательные параметр при повторном запросе
* dir - 0 только в один конец, 1 - туда-обратно
* tfl - тип поезда (3 - поезда и электрички, 2 - электрички, 1 - поезда)
* checkSeats поиск только с билетами (1 - с билетами, 0 - все поезда)
* code0 - код станции отправления
* code1 - код станции прибытия
* dt0 - дата отправления
* dt1 - дата возвращения

Ответы точно такие же как и в методе trainRoutes, только содержит 2 массива, в первом - туда, во-втором - обратно

### trainCarriages - получает список вагонов, свободные места, схема вагона, стоимость билетов, тип и класс обслуживания
![Вагоны](https://github.com/visavi/rzd-api/blob/master/screens/trainCarriages.png)

Необязательные параметр при повторном запросе
* dir - 0 только в один конец, 1 - туда-обратно
* code0 - код станции отправления
* code1 - код станции прибытия
* dt0 - дата отправления (28.03.2016)
* time0 - время отправления (15:30)
* tnum0 - номер поезда (072Е)

Возвращает следующий массив вагонов
* Стандартный ответ из запросов выше
* cnumber - номер вагона
* type - тип вагона
* typeLoc - полное наименование (Плацкартный, СВ, Купе, Люкс)
* clsType - 2Л, 2Э
* tariff - стоимость билета
* tariffServ - сервис сбор

* seats - массив мест (верхние, верхние боковые, нижние, нижние боковые итд)
* seats.*.places - список свободных мест
* seats.*.tariff - цены за место
* seats.type - сокр. наименование мест (up)
* seats.free - количество мест
* seats.label - полное наименование мест (Верхние)

* schemes схемы вагонов
* html - json массив информация о схеме вагонов
* image - ссылка на картинку

* insuranceCompany - массив с компаниями страхователями и правилами страхования
* shortName - наименование организации
* offerUrl - ссылка на файл с правилами, обычно PDF файл

### trainStationList - получение списка всех станций в текущем маршруте движения
![Станции](https://github.com/visavi/rzd-api/blob/master/screens/trainStationList.png)

Пример запроса https://pass.rzd.ru/ticket/services/route/basicRoute?STRUCTURE_ID=704&trainNumber=054Г&depDate=13.03.2016

Принимает параметры
Необязательные параметр при повторном запросе
* trainNumber - номер поезда 054Г
* depDate - дата отправления 13.03.2016

Возвращает следующий массив станций
* Station - название станции
* Code - код станции
* ArvTime - время прибытия
* WaitingTime - время стоянки
* DepTime - время отправления
* Distance - пройденная дистанция

### stationCode - Получение списка кодов станций

Принимает параметры
* stationNamePart - часть названия станции, минимум 2 символа
* compactMode - по умолчанию 'y'

Возвращает массив найденных данных
* station - имя станции
* code - код станции

К примеру при значении stationNamePart = 'ЧЕБ' будут возващены все станции начинающиеся на ЧЕБ (11 станций)

### License

The class is open-sourced software licensed under the [MIT license](http://opensource.org/licenses/MIT)
