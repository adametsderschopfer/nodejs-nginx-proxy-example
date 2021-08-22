# Сервер в связке Nginx + NodeJs

> Данная пошаговая инструкция поможет освоить основы на простом примере

**Для справки**

Сервер поднимался на `Debian 8` c характеристиками:

CPU - 1 ядро x 500 МГц

RAM - 512 МБ

Диск - 5 ГБ SSD+HDD

## Принцип работы на пальцах

`Nginx` будет отдавать статические файлы самостоятельно, динамический контент передавать из `NodeJS`.

## Установка и настройка Nginx

Представим что у вас чистый сервер и ничего не установлено. Идем в папку **/root** и становим `Nginx`:

```bash
$ apt-get install -y nginx
```

После установки `Nginx`, в папке **/var** появилась папка **/www**, а в ней папка **/html**, а в ней файл **index.html**.
Идем в папку **/var** и переименуем папку **/html** в папку **/nginx**:

```bash
$ mv /var/www/html /var/www/nginx
```

Создаем доп.файл **style.css** (для теста):

```bash
$ touch /var/www/nginx/style.css
```

В файле **index.html** пишем код, этот файл будет заглушкой:

```html
<h1>Заглушка</h1>
```

В файле **style.css** пишем код:

```css
* {background: #000;}
```

Еще нам нуно создать папку для `NodeJS`:

```bash
$ mkdir /var/www/nodejs
```

Далее прописывем на всякий случай права для папок:

```bash
$ chown www-data /var/www && chown www-data /var/www/nginx && chown www-data /var/www/nodejs
```

Теперь самое интересное, настраиваем файл конфига `Nginx`, редактируем файл **default**:

```bash
$ mcedit /etc/nginx/sites-available/default
```

Очищаем весь файл и пишем (комменты # ниже):
```
# Настройка сервера
server {
	# Nginx слушает порт 80
	# default_server - указан в /etc/nginx/nginx.conf
	listen 80 default_server;
	# Указываем "динамическую" папку NodeJS
	root /var/www/nodejs;
	# Указываем основной файл заглушки
	index index.html;
	# Устанавливаем страницы ошибок
	# В папке /var/www/errors должны быть файлы 
	# 50x.html и 40x.html соответственно
	error_page 500 502 503 504 /50x.html;
	error_page 400 401 402 403 404 /40x.html;
	location = /50x.html { 
		root /var/www/errors;
	}
	location = /40x.html { 
		root /var/www/errors;
	}
	# Указываем IP адрес сервера
	server_name IP_адрес_сервера;
	# Если мы обращаемся по любому УРЛ начиная с /
	# то сервер будет обрабатывать NodeJS
	location / {
		# Тут указываем IP|Url и порт (8000) для NodeJS
		# поскольку Nginx будет висеть на 80 порту
		proxy_pass http://IPorURL_адрес_сервера:8000;
		proxy_set_header Host $host;
	}
	# Если мы обращемся по УРЛ начинающийся с /nginx/
	# то мы будем подгружать "статичные" файлы хранящиеся в нем
	# в соответствии с наличием этих файлов в этой папке
	location /nginx/ {
		# Указываем корень
		root /var/www/;
		autoindex off;
		# Итого путь для Nginx будет
		# /var/www/static/
	}
}	 
```

Добавляем `Nginx` в автозагрузку и запускаем, что бы изменения применились, после проверяем статус:

```bash
$ systemctl enable nginx && systemctl start nginx && systemctl status nginx
```

## Установка и настройка NodeJS

Идем в папку **/root** и под пользователем `root` устанавливаем `cURL`

```bash
$ apt-get install -y curl
```

С помощью cURL скачиваем `NodeJS`, в моем случае верся 6:

```bash
$ curl -sL https://deb.nodesource.com/setup_6.x -o nodesource_setup.sh
```

Запускаем скаченный файл:

```bash
$ bash nodesource_setup.sh
```

Устанавливаем `NodeJS`

```bash
$ apt-get install -y nodejs build-essential
```

Готово! Можно протестировать:

```bash
$ node
> console.log ('hello world')
```

Вместе с `NodeJS` установился и `NPM` (Node Package Manager), с помощью которого мы установим `express` и `pm2`:

> С помощью демона `pm2` можно позабыть о проблемах с падением `NodeJS` (устанавливаем глобально):

```bash
$ npm install pm2 -g
```

> Инициализируем проект, создаем `package.json` в который будем фиксировать нужные пакеты (спасибо [@niiu](https://github.com/niiu) за подсказку)

```bash
$ npm init
```

> С помощью библиотеки `express` код будет писаться намного проще и быстрее (устанавлиаем локально):

```bash
$ cd /var/www/nodejs/
$ npm install express --save
```

Создаем файл **server.js** для `NodeJS`, который будет основным (входным) файлом:

```bash
$ touch /var/www/nodejs/server.js
```

Код файла **server.js** описан ниже:

```js
  // Настройки
  const setup = {port:8000}
  // Подключаем express
  const express = require ('express'); 
  // создаем приложение
  const app = express ();
  // Маршрутизируем GET-запрос http://ваш_сайт/test
  app.get('/test', (req, res) => {    
    res.send('Тест'); 
  });
  // Слушаем порт и при запуске сервера сообщаем
  app.listen(setup.port, () => {
    console.log('Сервер: порт %s - старт!', setup.port);
  });
```

Теперь можно добавить демону 1 процесс и запустить наш `NodeJS` сервер:

```bash
$ pm2 start /var/www/nodejs/server.js
```

> При этом у нас запущен сервер `Nginx`

После перезагрузки ОС, pm2 сам себя не запустит и соответственно не запустит процессы. Выполняем команды:

1. Сначала добавляем нужный процесс (в нашем случае скрипт `NodeJS`)
2. Потом сохраняем конфигурацию
3. После, добавляем `PM2` в сервисы ОС

```bash
$ pm2 start server.js
$ pm2 save
$ pm2 startup
```

## Готово

Если все запустилось, значит у вас ровные руки, а у меня талант писать пошаговые инструкции :)

**Тестируем**

Переходим на [http://IP_адрес_сайта:80/](http://IP_адрес_сайта:80/) - дожны увидеть фразу "Тест"

Переходим на [http://IP_адрес_сайта:80/nginx/style.css](http://IP_адрес_сайта:80/nginx/style.css) - дожны увидеть код стилей

Переходим на [http://IP_адрес_сайта:80/nginx/](http://IP_адрес_сайта:80/nginx/) или [http://IP_адрес_сайта:80/nginx/index.html](http://IP_адрес_сайта:80/nginx/index.html) - дожны увидеть заглушку

**Итого**

`Nginx` является прокси-сервером, `NodeJS` основным приложением. Первый висит на *80* порту, второй на *8000* и слушает первый. `NodeJS` отдает динамику, а `Nginx` отвечает за статику.  

> Если что-то не получилось или вы нашли ошибку, пишите в комментариях ниже!

## Полезные материалы

| Сайт | GitHub |
| --- | --- |
| [https://nginx.org/](https://nginx.org/) | [https://github.com/nginx/nginx](https://github.com/nginx/nginx) |
| [https://nodejs.org/](https://nodejs.org/) | [https://github.com/nodejs/node](https://github.com/nodejs/node) |
| [http://expressjs.com/](http://expressjs.com/) | [https://github.com/expressjs/express](https://github.com/expressjs/express) |
| [https://www.npmjs.com/](https://www.npmjs.com/) | [https://github.com/npm](https://github.com/npm) |
| [https://www.npmjs.com/package/pm2](https://www.npmjs.com/package/pm2) |  |
| [https://packages.debian.org/ru/jessie/curl](https://packages.debian.org/ru/jessie/curl) |  |