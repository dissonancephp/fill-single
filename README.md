# Dissonance Web (MVP, Single-file)

## Описание

Фреймворк создан для небольших независимых приложений, 
которые могут расширяться плагинами. 
Фреймворк оптимизирован для работы с большим количеством приложений, а также для работы в качетсве подсистемы для основного фреймворка. 

Каждое приложение является композер пакетом, 
с дополнительным описанием прямо в файле composer.json.

## Характеристики
- PSR дружественный
- Мало зависимостей (только PSR интерфейсы и PSR-7 имплементация)
- Небольшой вес (370 кб дев версия и формтированием и коментариями, продакт 170 кб)
- Оптимизирован для работы в симбиозе с другими фреймворками
- Многоуровневая система контейнеров (Ядро<-Приложение<-Плагин), с доступом к контейнеру родителю.
- Виртуальная файловая система (прокидывание статики прямо из папки пакета)
- Всем знакомое апи контейнера
- Шаблонизатор Blade(урезанный и кривой пока), + возможность прокинуть свой шаблонизатор.
- Никаких сборщиков статики (Каждый пакет должен иметь уже скомпилировынные файлы).
- Отложенный роутинг (грузятся только роуты запрошенного приложения, определяется по префиксу-поселению).
- Возможность расширять конревые сервисы (Бутстраперы и сервисы).
- У каждого приложения свой сервис контейнер и сервисы.
- Поддержка кеша (PSR-16 Simple Cache) + Кешируемый cервис контейнер.
- Для тех, кто будет тестировать: отличное приключение (почти реверс), абсолютно без документации и все одном файле!!!)))

## Установка
```
composer require dissonance/full-single 
```

## Запуск

Фреймворк подключается из композера прямо в ваш index.php.
 
 Если вы используете уже фреймворк, то необходимо включить режим симбиоза в конфиге
```php
$config['symbiosis'] = true;
```
##### Инициализация
```php

$basePath = dirname(__DIR__);// корневая папка проекта
include_once $basePath. '/vendor/autoload.php';

$config  = [
    'debug' => true,
    'symbiosis' => true, // Режим симбиоза, если включен и фреймворк не найдет обработчик,
    // то он ничего не вернет и основной фреймворк смодет сам обработать запрос
    'default_host' => 'localhost',// для консоли , но ее пока нет
    'uri_prefix' => 'dissonance', // Префикс в котором работет фреймворк, если пустой то работае от корня
    'base_path' => $basePath, // базовая папка проекта
    'assets_prefix' => '/assets',
    'storage_path' =>  $basePath . '/storage', // Если убрать то кеш отключится
    'bootstrappers' => [
        \Dissonance\Develop\Bootstrap\DebugBootstrap::class,/// debug only
        \Dissonance\Bootstrap\EventBootstrap::class,
        \Dissonance\SimpleCacheFilesystem\Bootstrap::class,
        \Dissonance\PackagesLoaderFilesystem\Bootstrap::class,
        \Dissonance\Packages\PackagesBootstrap::class,
        \Dissonance\Packages\ResourcesBootstrap::class,
        \Dissonance\Apps\Bootstrap::class,
        \Dissonance\Http\Bootstrap::class,
        \Dissonance\HttpKernel\Bootstrap::class,
        \Dissonance\CacheRouting\Bootstrap::class,
        \Dissonance\ViewBlade\Bootstrap::class,
    ],
    'providers' => [
        \Dissonance\Http\Cookie\CookiesProvider::class,
        \Dissonance\SettlementsRouting\Provider::class,
        \Dissonance\Session\NativeProvider::class,
    ],
    'providers_exclude' => [
        \Dissonance\Routing\Provider::class,
    ]
];

// Базовая постройка контейнера
$core = new \Dissonance\Core($config);
// Или через билдер с кешем
$cache = new Dissonance\SimpleCacheFilesystem\SimpleCache($basePath . '/storage/cache/core');
$core = (new \Dissonance\CachedContainer\ContainerBuilder($cache))
    ->buildCore($config);

// Запуск 
$core->run();
// Дальше может идти код инициализации и отработки другого фреймворка...

```
##### Схема описания расширения и приложения для фреймворка
Берем стандартный пакет композера и добавляем:
```json
{
  "name": "vendor/package",
  "require": {
   // ...
  },
  "autoload": {
   ///
  
  },
// Добавляем описание пакета для фреймворка
  "extra": {
    "dissonance": {
          "id": "wso.my_package_id", // ID пакета формируется на сайте фреймворка, но можно локально любой ставить
           // Описание приложения, пакет может и не иметь секцию приложения, а быть лишь расширением
          "app": { 
                "id": "my_package_id", // Id приложения, указывается без префикса родительского приложения
                "parent_app": "wso", // ID родительсского приложения, если приложение плагин 
                "name": "WSO Users exporter", // Имя приложения, используется в списке приложений и меню
                "routing": "\\\\MyVendor\\\\MySuperPackage\\\\Routing", // Класс роутинга, не обязательно
                "controllers_namespace": "\\\\Dissonance\\\\Develop\\\\Controllers", // Базовый неймспейс для контроллеров, не обязательно
                "version": "1.0.0", // Версия, не обязательно, плагины могут проверять и подстаиваться под изменения
                "providers": [ // Провайдеры приложения, не обязательно
                  "MyVendor\\\\MySuperPackage\\\\Providers\\\\AppProvider"
                ],
                // Не обязательно! Наследник от \\Dissonance\\App\\Application
                "app_class": "MyVendor\\\\MySuperPackage\\\\MyAppContainer" 
          },
    
          // Расширения ядра фреймворка, не обязательно
          "bootstrappers":[
             "MyVendor\\\\MySuperPackage\\\\CoreBootstrap" // Загрузчики
          ],
          "providers" : [
             "MyVendor\\\\MySuperPackage\\\\MyDbProvider" // Провайдеры
          ],
          "providers_exclude" : [
              // Исключение провайдеров из загрузки
              // Например при двух пакетах одной библиотеки позволяет исключить не нужную
          ]     
    }
  }
}

```

#### Пример пакета только со статикой
Всего пару строк:
```json
{
  "name": "vendor/package",
  "require": {
   // ...
  },
  "autoload": {
   // ...
  },
  "extra": {
    "dissonance": {
          "id": "my_super_theme_2",
          // Можно указать что то одно или все вместе
          "public_path": "assets", // Папка со статикой, относительно корня пакета 
          "resources_path": "my_resources", // Папка c шаблонами и другими файлами, не доступны через http
          // можно прокинуть в веб при необходимости через специальный объект доступа к ресурсам
    }
  }
}

```


#### Пример пакета приложения
При конфигурации приложения можно не указывать пути для статики и ресурсов, тогда будут определены пути по умолчанию:

- public_path = assets
- resources_path = resources

Шаблоны всегда дожны лежать в директории /view/ в папке ресурсов!
```json
{
  "name": "vendor/package",
  "require": {
   // ...
  },
  "autoload": {
   // ...
  },
  "extra": {
    "dissonance": {
           "app": { 
                "id": "my_package_id", // Id приложения
                "routing": "\\\\MyVendor\\\\MySuperPackage\\\\Routing",
                "controllers_namespace": "\\\\Dissonance\\\\Develop\\\\Controllers"
          },
    }
  }
}

```

## Примерная структура пакета
Четкой обязательной структуры нет, можно использовать любую. 
```text
vendor/
   -/my_vendor
      -/my_package_name
           -/assets          - Статика
                -/js
                -/css
                -/...
           -/resources       - Ресурсы
                -/views
                -/...
           -/src             - Ваш пакет
               -/Http
                   -/Cоntrollers
                   -/...
               -/ ...
               -/Routing.php
          -/composer.json
```

При необходимости можно поселить все классы для приложения фреймворка в подпапку src/Dissonance. Так не будет путаницы с функционалом вашего пакета.

```text
vendor/
   -/my_vendor
      -/my_package_name
           -/dissonance
                   -/assets          - Статика
                        -/js
                        -/css
                        -/...
                   -/resources       - Ресурсы
                        -/views
                        -/...
           -/src                     - Ваш пакет
               -/Dissonance
                       -/Http
                           -/Cоntrollers
                           -/...
                       -/Routing.php
              -/Ваши папки и файлы ...
              
          -/composer.json
```


