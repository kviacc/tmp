## Как поднять проект

Для работы с проектом вам потребуется:

- PHP 8.0
- Node JS 14+
- composer (https://getcomposer.org/)
- Git
- Supervisor
- Nginx
- MySQL 8
- Redis

### Установка всего необходимого

Склонируйте проект. После этого установите все зависимости PHP и JS:
```bash
composer install
npm i
```
Соберите фронт:
```bash
npm run prod
```

### Конфигурация проекта
Конфигурирование проекта выполняется стандартно через файл .env. Его можно создать на основе файла .env.example, там
необходимо указать логин и пароль для доступа к базе данных, а так же необходимо указать адреса смарт-контрактов 
платформы и RCL, адреса сервисов для проведения транзакций и проверки результатов и их ключи, номер сети эфира, в которую
выложен смарт-контракт:
```
CHAIN_ID=4
RECYCLING_TOKEN_CONTRACT=address
RECYCLING_PLATFORM_CONTRACT=address

ETHERSCAN_API_ADDRESS=https://api-rinkeby.etherscan.io/api
ETHERSCAN_API_KEY=key
ETHERSCAN_BASE=https://rinkeby.etherscan.io/address/
```

После заполнения файла для создания базы данных нужно выполнить команду
```
php artisan migrate
```

### Настройка nginx

Используется комбинация nginx для поднятия статики и php-fpm для запуска php. Текущая конфигурация nginx:
```
server {
    # При установке в другой каталог замените путь. Но он должен указывать на папку public проекта
	root /var/www/Recycle/public;

	index index.php index.html index.htm index.nginx-debian.html;

	server_name recykl.ru;

	location / {
	    # Для правильной работы все пути, которым не соответствуют файлы и папки, должны быть перенаправлены
	    # на обработку в файл index.php
		try_files $uri $uri/ /index.php?$query_string;
	}

	# Задаём обработку php-файлов через php-fpm
	#
	location ~ \.php$ {
		include snippets/fastcgi-php.conf;
	
		# With php-fpm (or other uix sockets):
	    fastcgi_pass unix:/var/run/php/php8.0-fpm.sock;
	}

	listen 80;
	listen [::]:80;
}
```
### Настройка Cron

Необходимо настроить ежеминутный запуск шедулера Laravel. Для этого введите команду ```crontab -e``` и введите в редакторе следующее

```
* * * * * export LOG_CHANNEL="cli"; php /var/www/Recycle/artisan schedule:run >> /dev/null 2>&1

```
Обратите внимание на пустую строку в конце - она обязательна, иначе крон не примет файл

### Настройка Supervisor

В системе используются очереди, воркеры которых должны работать в фоновом режиме. Для этого они запускаются через
менеджер процессов Supervisor, который, в том числе, будет обеспечивать автоматический рестарт воркеров в случае падения
по какой-либо причине. В системе используется как стандартный воркер очередей Laravel, который может быть запущен в любом количестве
экземпляров, так и кастомный воркер внутренних транзакций, который должен быть запущен строго в одном экземпляре, иначе
система начнёт давать сбои при вызовах API.

Конфигурация для стандартного воркера (в данном лучае запускается два экземпляра)[ /etc/supervisor/conf.d/recycle.conf ]

```
[program:recycle]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/Recycle/artisan queue:work --timeout 580
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=root
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/Recycle/worker.log
stopwaitsecs=3600
environment=LOG_CHANNEL="cli"
```

Конфигурация воркера внутренних транзакций [ /etc/supervisor/conf.d/recycle-internal.conf ]:
```
[program:recycle-internal]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/Recycle/artisan internal-transactions:queue
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=root
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/Recycle/worker-internal.log
stopwaitsecs=3600
environment=LOG_CHANNEL="cli"
```

Данные конфигурации нужно сохранить в файлах ```/etc/supervisor/conf.d/recycle.conf``` и ```/etc/supervisor/conf.d/recycle-internal.conf``` соответственно.
После этого выполнить следующие команды:
```bash
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start recycle:*
sudo supervisorctl start recycle-internal:*
```
В случае внесения изменений в проект, касающихся задач, необходимо перезапустить данные воркеры:
```bash
sudo supervisorctl restart recycle:*
sudo supervisorctl restart recycle-internal:*
```

<p align="center"><a href="https://laravel.com" target="_blank"><img src="https://raw.githubusercontent.com/laravel/art/master/logo-lockup/5%20SVG/2%20CMYK/1%20Full%20Color/laravel-logolockup-cmyk-red.svg" width="400"></a></p>

<p align="center">
<a href="https://travis-ci.org/laravel/framework"><img src="https://travis-ci.org/laravel/framework.svg" alt="Build Status"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://img.shields.io/packagist/dt/laravel/framework" alt="Total Downloads"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://img.shields.io/packagist/v/laravel/framework" alt="Latest Stable Version"></a>
<a href="https://packagist.org/packages/laravel/framework"><img src="https://img.shields.io/packagist/l/laravel/framework" alt="License"></a>
</p>

## About Laravel

Laravel is a web application framework with expressive, elegant syntax. We believe development must be an enjoyable and creative experience to be truly fulfilling. Laravel takes the pain out of development by easing common tasks used in many web projects, such as:

- [Simple, fast routing engine](https://laravel.com/docs/routing).
- [Powerful dependency injection container](https://laravel.com/docs/container).
- Multiple back-ends for [session](https://laravel.com/docs/session) and [cache](https://laravel.com/docs/cache) storage.
- Expressive, intuitive [database ORM](https://laravel.com/docs/eloquent).
- Database agnostic [schema migrations](https://laravel.com/docs/migrations).
- [Robust background job processing](https://laravel.com/docs/queues).
- [Real-time event broadcasting](https://laravel.com/docs/broadcasting).

Laravel is accessible, powerful, and provides tools required for large, robust applications.

## Learning Laravel

Laravel has the most extensive and thorough [documentation](https://laravel.com/docs) and video tutorial library of all modern web application frameworks, making it a breeze to get started with the framework.

If you don't feel like reading, [Laracasts](https://laracasts.com) can help. Laracasts contains over 1500 video tutorials on a range of topics including Laravel, modern PHP, unit testing, and JavaScript. Boost your skills by digging into our comprehensive video library.

## Laravel Sponsors

We would like to extend our thanks to the following sponsors for funding Laravel development. If you are interested in becoming a sponsor, please visit the Laravel [Patreon page](https://patreon.com/taylorotwell).

### Premium Partners

- **[Vehikl](https://vehikl.com/)**
- **[Tighten Co.](https://tighten.co)**
- **[Kirschbaum Development Group](https://kirschbaumdevelopment.com)**
- **[64 Robots](https://64robots.com)**
- **[Cubet Techno Labs](https://cubettech.com)**
- **[Cyber-Duck](https://cyber-duck.co.uk)**
- **[Many](https://www.many.co.uk)**
- **[Webdock, Fast VPS Hosting](https://www.webdock.io/en)**
- **[DevSquad](https://devsquad.com)**
- **[Curotec](https://www.curotec.com/services/technologies/laravel/)**
- **[OP.GG](https://op.gg)**
- **[CMS Max](https://www.cmsmax.com/)**
- **[WebReinvent](https://webreinvent.com/?utm_source=laravel&utm_medium=github&utm_campaign=patreon-sponsors)**

## Contributing

Thank you for considering contributing to the Laravel framework! The contribution guide can be found in the [Laravel documentation](https://laravel.com/docs/contributions).

## Code of Conduct

In order to ensure that the Laravel community is welcoming to all, please review and abide by the [Code of Conduct](https://laravel.com/docs/contributions#code-of-conduct).

## Security Vulnerabilities

If you discover a security vulnerability within Laravel, please send an e-mail to Taylor Otwell via [taylor@laravel.com](mailto:taylor@laravel.com). All security vulnerabilities will be promptly addressed.

## License

The Laravel framework is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).
