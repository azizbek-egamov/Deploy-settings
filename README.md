# Документация по развертыванию Django с Nginx и Gunicorn

Это руководство описывает, как развернуть Django-проект с использованием Gunicorn в качестве WSGI-сервера и Nginx в качестве обратного прокси.

---

## Предпосылки

- Рабочий Django-проект.
- Установленные Gunicorn и Nginx.
- Настроенные DNS для домена (например, `yourdomain.com`).

---

## Установка и запуск Gunicorn

Установите Gunicorn:

```bash
pip install gunicorn
```

Примечание:

- project_name — название проекта.
- database_name — имя базы данных.
- user_name — имя пользователя базы данных.
- password — пароль базы данных.
- sammy — имя пользователя Ubuntu (замените на ваше).
- Важно!
- Не рекомендуется выполнять эти действия от имени root. Создайте пользователя с правами sudo (см. инструкцию).

Структура проекта может выглядеть следующим образом:

```bash
├── projectname
│   ├── appname
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── projectname
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   ├── templates
│   ├── static
│   ├── media
│   ├── manage.py
│   └── requirements.txt
```

Установка необходимых пакетов
Обновите пакеты и установите необходимые инструменты:

```bash
sudo apt update
sudo apt upgrade
sudo apt install -y python3-pip python-dev-is-python3 libpq-dev vim git htop postgresql postgresql-contrib nginx redis-server python3.12-venv
```

Настройка PostgreSQL
1. Подключитесь к PostgreSQL от имени пользователя postgres:

```bash
sudo -u postgres psql
```

2. Создайте базу данных:

```bash
CREATE DATABASE database_name;
```

3. Создайте пользователя базы данных:

```bash
CREATE USER user_name WITH PASSWORD 'password';
```

4. Настройте параметры пользователя:

```bash
ALTER ROLE user_name SET client_encoding TO 'utf8';
ALTER ROLE user_name SET default_transaction_isolation TO 'read committed';
ALTER ROLE user_name SET timezone TO 'UTC';

```

5. Предоставьте пользователю все привилегии на базу данных:

```bash
GRANT ALL PRIVILEGES ON DATABASE database_name TO user_name;
```

6. Выйдите из PostgreSQL:

```bash
\q
```

Клонирование проекта из Git-репозитория
Клонируйте ваш проект:

```bash
git clone your_repository_link
```

Настройка виртуального окружения
1. Перейдите в каталог проекта:

```bash
cd my_project
```

2. Создайте виртуальное окружение:

```bash
python3 -m venv venv
```

3. Активируйте виртуальное окружение и установите зависимости:

```bash
. venv/bin/activate && pip3 install -r requirements.txt
```

Если проект использует дополнительные пакеты, добавьте их в requirements.txt.

Настройка Django-проекта
1. Редактирование settings.py:

- Укажите домен и IP-адреса в списке ALLOWED_HOSTS:

```bash
ALLOWED_HOSTS = ['your_server_domain_or_IP', 'second_domain_or_IP']
```
- Настройте подключение к базе данных:

```bash
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'database_name',
        'USER': 'user_name',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```
- Задайте путь для статических файлов:

```bash
STATIC_ROOT = BASE_DIR / 'staticfiles'
```

2. Применение миграций:

```bash
python3 manage.py makemigrations
python3 manage.py migrate
```

3. Создание суперпользователя:

```bash
python3 manage.py createsuperuser
```

4. Сбор статических файлов:

```bash
python3 manage.py collectstatic
```

5. Проверка работы сервера:

```bash
python3 manage.py runserver 0.0.0.0:8000
```

Откройте в браузере http://server_ip:8000 для проверки работы.
Важно: В продакшене значение DEBUG должно быть установлено в False, чтобы не раскрывать подробности об ошибках пользователям.

Тестирование Gunicorn
Проверьте запуск проекта через Gunicorn:

```bash
cd ~/myproject
gunicorn --bind 0.0.0.0:8000 myproject.wsgi
```

После проверки остановите Gunicorn (нажмите Ctrl+C) и деактивируйте виртуальное окружение:

```bash
deactivate
```

Настройка Gunicorn с systemd
Для автоматического управления Gunicorn создайте сервисный файл:

1. Откройте файл для редактирования:

```bash
sudo nano /etc/systemd/system/gunicorn.service
```

2. Вставьте следующее содержимое (отредактируйте пути и имя пользователя):

```bash
# Defoult
[Unit]
Description=Ardent ASGI Server
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/root/Api_Ardent
ExecStart=/home/sammy/myproject/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/sammy/myproject/myproject.sock myproject.wsgi:application

[Install]
WantedBy=multi-user.target



# Uvicorn
Group=root

Environment="DJANGO_SETTINGS_MODULE=set_app.settings"
ExecStart=/root/Api_Ardent/venv/bin/gunicorn --workers 3 --bind unix:/root/Api_Ardent/set_app/set_app.sock set_app.asgi:application -k uvicorn.workers.UvicornWorker
```

2.1. Telegram bot:

```bash
[Unit]
Description=Runbot Django Command
After=network.target

[Service]
User=root
WorkingDirectory=/root/project
ExecStart=/root/project/venv/bin/python manage.py runbot
Restart=always

[Install]
WantedBy=multi-user.target
```

3. Сохраните файл (Ctrl+O, Enter) и выйдите (Ctrl+X).
4. Примените изменения и запустите сервис:

```bash
sudo systemctl daemon-reload
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```

5. Проверьте статус сервиса:

```bash
sudo systemctl status gunicorn
```

6. Для просмотра логов используйте:

```bash
sudo journalctl -u gunicorn
```

7. Если вносите изменения в файл сервиса, не забудьте выполнить:

```bash
sudo systemctl daemon-reload 
sudo systemctl restart gunicorn
```

При успешном запуске в каталоге проекта появится файл myproject.sock

Настройка Nginx
1. Создайте конфигурационный файл для вашего проекта:

```bash
sudo nano /etc/nginx/sites-available/myproject
```

2. Вставьте следующую конфигурацию (отредактируйте пути и имя домена/IP):

```bash
# Defoult
server {
    server_name server_domain_or_IP;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        alias /home/sammy/myproject/staticfiles/;
    }

    location /media/ {
        alias /home/sammy/myproject/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/sammy/myproject/myproject.sock;
    }
}

# How need you
server {
    listen 443 ssl http2;
    server_name name_site.uz www.name_site.uz;

    ssl_certificate /etc/letsencrypt/live/name_site.uz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/name_site.uz/privkey.pem;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/project_name/set_app.sock;
        proxy_read_timeout 60;
        proxy_connect_timeout 60;
        proxy_send_timeout 60;
        client_max_body_size 100M;

        set $cors_origin "";
        if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
            set $cors_origin $http_origin;
        }

        add_header 'Access-Control-Allow-Origin' $cors_origin always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE' always;
        add_header 'Access-Control-Allow-Headers' 'Origin, Content-Type, Authorization, X-CSRFToken' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;

        if ($request_method = OPTIONS) {
            return 204;
        }
    }

    location /static/ {
        alias /root/Api_Ardent/staticfiles/;
        set $cors_origin "";
        if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
            set $cors_origin $http_origin;
        }
        add_header 'Access-Control-Allow-Origin' $cors_origin always;
    }

    location /media/ {
        alias /root/Api_Ardent/media/;
        set $cors_origin "";
        if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
            set $cors_origin $http_origin;
        }
        add_header 'Access-Control-Allow-Origin' $cors_origin always;
    }
}


```

3. Активируйте сайт, создав символическую ссылку:

```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
```

4. Проверьте синтаксис конфигурации Nginx:
5. Перезапустите Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

6. Дать права доступа

```bash
chmod g+x /root/
chmod g+r /root/
sudo chgrp www-data /root/

chmod g+x /root/
chmod g+r /root/
sudo chgrp www-data /root/
```

Настройка HTTPS и исправление CORS в Nginx

Настройка HTTPS с Let's Encrypt

Для обеспечения безопасности соединения используется сертификат Let's Encrypt. В конфигурации Nginx добавлены следующие строки:

```bash
server {
    listen 80;
    server_name testpost.uz www.testpost.uz;
    return 301 https://$host$request_uri;  # Авто-редирект на HTTPS
}

server {
    listen 443 ssl;
    server_name testpost.uz www.testpost.uz;

    ssl_certificate /etc/letsencrypt/live/testpost.uz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/testpost.uz/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        alias /root/test_post/staticfiles/;
    }

    location /media/ {
        alias /root/test_post/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/root/test_post/testpost.sock;
    }
}
```
Эти параметры указывают на путь к SSL-сертификату и закрытому ключу.

Как получить Сертификат HTTPS:
Если сертификат отсутствует, его можно получить с помощью Certbot:
 🔹 Шаг 1: Установка Certbot
 
```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

 🔹 Шаг 2: Получение SSL-сертификата

 ```bash
sudo certbot --nginx -d name_site.com -d www.name_site.com
 ```

 🔹 Шаг 3: Получение SSL-сертификата
```bash
sudo certbot --nginx -d crmbot.uz -d www.crmbot.uz --agree-tos --register-unsafely-without-email --non-interactive -v
```

🔹 Шаг 3: Проверка и автоматическое обновление
Запусти тестовый запуск обновления:

Для автоматического обновления сертификата:

```bash
sudo certbot renew --dry-run
```

Если все ок, Certbot сам будет обновлять сертификат.

Исправление CORS в Nginx

Для разрешения запросов с определённых доменов используется настройка CORS. В конфигурации Nginx реализован динамический механизм установки заголовка Access-Control-Allow-Origin.

Нужно Утановить
```bash
pip install django-cors-headers
```

в settings.py
```bash

INSTALLED_APPS = [
    ...,
    'corsheaders',
    ...
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',# ← ОБЯЗАТЕЛЬНО первым!
    'django.middleware.common.CommonMiddleware',
    ...
]
```

Доступ для всех
```bash
CORS_ALLOW_ALL_ORIGINS = True
CORS_ALLOW_CREDENTIALS = True

CORS_ALLOW_HEADERS = [
    "accept",
    "accept-encoding",
    "authorization",
    "content-type",
    "dnt",
    "origin",
    "user-agent",
    "x-csrftoken",
    "x-requested-with",
]

CORS_ALLOW_METHODS = [
    "DELETE",
    "GET",
    "OPTIONS",
    "PATCH",
    "POST",
    "PUT",
]
```

Органичение доступа
```bash
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",
    "https://frontend-site.com"
]
CORS_ALLOW_CREDENTIALS = True
```

Также в Media
```bash
location /media/ {
    alias /root/test/media/;
    add_header Access-Control-Allow-Origin *;
}
```
после надо перезапустить ngx

Для проверки Cors
```bash
curl -i https://name_domen.uz/name_api/
```

Допольнительно провека Cors
```bash
curl -i https://name_domen.uz/name_api/ -H "Origin: http://localhost:3000"
```

После нужно перезапустить
```bash
sudo systemctl restart gunicorn
```

Должно вывести 
- Access-Control-Allow-Origin: *
- Access-Control-Allow-Origin: http://localhost:3000

Основные CORS-настройки:

```bash
set $cors_origin "";
if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
    set $cors_origin $http_origin;
}

add_header 'Access-Control-Allow-Origin' $cors_origin always;
add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE' always;
add_header 'Access-Control-Allow-Headers' 'Origin, Content-Type, Authorization, X-CSRFToken' always;
add_header 'Access-Control-Allow-Credentials' 'true' always;
```

Этот блок кода:
 1. Проверяет, совпадает ли источник запроса ($http_origin) с одним из разрешённых доменов.
 2. Если да, устанавливает Access-Control-Allow-Origin на $http_origin.
 3. Разрешает использование методов GET, POST, OPTIONS, PUT, DELETE.
 4. Разрешает заголовки Origin, Content-Type, Authorization, X-CSRFToken.
 5. Позволяет передавать куки (Access-Control-Allow-Credentials).

Обработка preflight-запросов (OPTIONS)

```bash
if ($request_method = OPTIONS) {
    return 204;
}
```
Этот блок кода отвечает статусом 204 No Content на preflight-запросы, которые браузер отправляет перед выполнением CORS-запросов.

CORS для статики и медиа

Дополнительно применяются CORS-настройки для статических и медиа-файлов:

```bash
location /static/ {
    alias /root/Api_Ardent/staticfiles/;
    set $cors_origin "";
    if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
        set $cors_origin $http_origin;
    }
    add_header 'Access-Control-Allow-Origin' $cors_origin always;
}

location /media/ {
    alias /root/Api_Ardent/media/;
    set $cors_origin "";
    if ($http_origin ~* (https://menu-three-green\.vercel\.app|https://waiter-app\.vercel\.app|https://waiter-app-six\.vercel\.app|http://localhost:5173|http://localhost:5174)) {
        set $cors_origin $http_origin;
    }
    add_header 'Access-Control-Allow-Origin' $cors_origin always;
}
```
Эти настройки позволяют фронтенд-приложениям запрашивать файлы из /static/ и /media/ без проблем с CORS.

# Defoult start

```bash
gunicorn --workers 3 --bind unix:/root/food/set_app/set_app.sock set_app.wsgi:application
```

# Install unicorn

```bash
pip install uvicorn
```

# Start unicorn

```bash
gunicorn --workers 3 --bind unix:/root/Api_Ardent/set_app/set_app.sock set_app.asgi:application -k uvicorn.workers.UvicornWorker
```

# Websocket

```bash
pip install channels
pip install daphne
```

```bash
# settings
ASGI_APPLICATION = 'your_project_name.asgi.application'

# asgi.py
import os
import django

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'set_app.settings')
django.setup() 

from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter
from django.urls import re_path
from channels.routing import ProtocolTypeRouter, URLRouter
from set_main import consumers

websocket_urlpatterns = [
    re_path(r"ws/chat/(?P<chat_id>\d+)/$", consumers.ChatConsumer.as_asgi()),
]

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": URLRouter(websocket_urlpatterns)
})

# set_mian routing.py
from django.urls import path
from . import consumers

websocket_urlpatterns = [
    path("ws/some_path/", consumers.MyConsumer.as_asgi()),
]

# set_main consumers.py
class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.chat_id = self.scope['url_route']['kwargs']['chat_id']
        self.room_group_name = f'chat_{self.chat_id}'

        # ❗ Без проверки пользователя — WebSocket работает анонимно
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )

    async def receive(self, text_data=None, bytes_data=None):
        data = json.loads(text_data)
        text = data.get('text', '')
        sender_id = data.get('sender_id')  # ⚠️ Т.к. scope["user"] нет, берём из сообщения

        if not sender_id:
            return

        message = await self.create_message(sender_id, text)

        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': {
                    'id': message.id,
                    'text': message.text,
                    'sender': sender_id,
                    'created_at': str(message.created_at)
                }
            }
        )

# gunicorn 
server {
    server_name building.ardentsoft.uz;

    client_max_body_size 50M;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        alias /root/building/staticfiles/;
        add_header Access-Control-Allow-Origin "https://localhost:5173" always;
        add_header Access-Control-Allow-Credentials "true" always;
    }

    location /media/ {
        alias /root/building/media/;
        add_header Access-Control-Allow-Origin "https://localhost:5173" always;
        add_header Access-Control-Allow-Credentials "true" always;
    }

    # ✅ WebSocket — проксируем на Daphne (ASGI)
    location /ws/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }

    # Основное Django-приложение (через unix socket)
    location / {
        include proxy_params;
        proxy_pass http://unix:/root/building/set_app/set_app.sock;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/building.ardentsoft.uz/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/building.ardentsoft.uz/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = building.ardentsoft.uz) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    server_name building.ardentsoft.uz;
    listen 80;
    return 404; # managed by Certbot
}
```

```bash
sudo nginx -t
sudo systemctl reload nginx
```

websocket run
```bash
DJANGO_SETTINGS_MODULE=set_app.settings daphne set_app.asgi:application
```

```bash
wss://building.ardentsoft.uz/ws/chat/1/
```

Websocket daphne with flatter 
```bash

# set_app/asgi.py
import os
import django

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'set_app.settings')
django.setup() 

from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter
from django.urls import re_path
from channels.routing import ProtocolTypeRouter, URLRouter
from set_main import consumers
from channels.auth import AuthMiddlewareStack

websocket_urlpatterns = [
    re_path(r"ws/chat/(?P<chat_id>\d+)/$", consumers.ChatConsumer.as_asgi()),
]

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(websocket_urlpatterns)
    ),
})


# ngxs
location /ws/ {
    proxy_pass http://unix:/root/building/set_app/set_app.sock;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400;
}

sudo nginx -t       # проверяем конфигурацию
sudo systemctl reload nginx

# запуск
daphne -u /root/building/set_app/set_app.sock set_app.asgi:application
daphne -b 127.0.0.1 -p 8001 set_app.asgi:application
```

# Start stop

```bash
ps aux | grep gunicorn
pkill -9 gunicorn
```

```bash
sudo rm /etc/nginx/sites-enabled/default
```

```bash
# Установи максимальный размер загружаемого файла, например, до 100MB
DATA_UPLOAD_MAX_MEMORY_SIZE = 104857600  # 100 * 1024 * 1024
FILE_UPLOAD_MAX_MEMORY_SIZE = 104857600

client_max_body_size 100M;
```

Завершение настройки
Откройте в браузере домен или IP-адрес сервера для проверки развертывания проекта. Рекомендуется также настроить SSL-сертификат (например, с помощью <a href="https://letsencrypt.org/">Let's Encrypt</a>) для обеспечения безопасности соединения.

Дополнительная информация

- <a href="https://docs.djangoproject.com/en/5.1/">Официальная документация Django</a>

Если возникнут вопросы или проблемы, обратитесь к сообществу или на <a href="https://stackoverflow.com/questions">Stack Overflow</a>.

Va nihoyat serverni sozlab boldik!!! Siz brauzeringizda server IP si yo'ki domenini kiritib buni tekshirib ko'rishingiz mumkin, ha aytkancha serveringiz uchun <a href="https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04">SSL sertifikat olish</a>ni unitmang

https://stackoverflow.com/questions/41951792/gunicorn-and-django-error-permission-denied-for-sock

Ba'zi holatlarda 502 xatolikini olishingiz mumkin: https://stackoverflow.com/questions/25774999/nginx-stat-failed-13-permission-denied

NGINX faylni o'zgartirish kerak rootga https://stackoverflow.com/questions/70111791/nginx-13-permission-denied-while-connecting-to-upstream
