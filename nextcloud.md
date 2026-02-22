# Установка Nextcloud на домашний Ubuntu Server (Dexp Compact M008)

Устанавливаем Nextcloud через Docker-контейнер — это удобно, потому что не нужно вручную настраивать Apache, PHP и MariaDB: всё уже упаковано в образы.

---

## Подготовка папок

Создаём директории для хранения данных приложения и базы данных:

```bash
mkdir -p ~/nextcloud/data ~/nextcloud/db
```

---

## docker-compose.yml

Переходим в редактор:

```bash
nano ~/nextcloud/docker-compose.yml
```

Содержимое файла:

```yaml
version: '3'

services:
  db:
    image: mariadb:10.6
    restart: always
    volumes:
      - ~/nextcloud/db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=пароль_для_root
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=пароль_для_nextcloud

  app:
    image: nextcloud:32.0.5
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ~/nextcloud/data:/var/www/html
    depends_on:
      - db
    environment:
      - MYSQL_HOST=db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=пароль_для_nextcloud
```

---

## Разбор docker-compose.yml

Этот файл — конфигурация для Docker Compose, инструмента, который запускает несколько контейнеров сразу, связывает их между собой и настраивает тома, порты и переменные окружения.

В файле два основных раздела: `version` и `services`.

### version: '3'

Указывает версию синтаксиса Docker Compose. Версия 3 — современная, поддерживает большинство возможностей. Это «шапка», которая говорит Docker, как читать файл.

### services

Здесь перечислены все контейнеры, которые нужно запустить. В нашем случае их два: `db` (база данных) и `app` (веб-приложение Nextcloud). Каждый контейнер изолирован и «думает», что он один на сервере.

---

### Сервис `db` — база данных MariaDB

```yaml
db:
  image: mariadb:10.6       # берёт готовый образ из Docker Hub
  restart: always           # автоматически перезапускается при сбое
  volumes:
    - ~/nextcloud/db:/var/lib/mysql   # слева — папка на сервере, справа — внутри контейнера
  environment:
    - MYSQL_ROOT_PASSWORD=пароль_для_root
    - MYSQL_DATABASE=nextcloud        # создаёт базу данных nextcloud
    - MYSQL_USER=nextcloud            # пользователь базы данных
    - MYSQL_PASSWORD=пароль_для_nextcloud
```

**Почему MariaDB?** Это open-source потомок MySQL с совместимостью на 99%, активной разработкой сообщества и официальным образом в Docker Hub. Nextcloud прямо рекомендует MariaDB в своей документации.

**Volumes** — том связывает папку на сервере с папкой внутри контейнера. Благодаря этому данные сохраняются даже если контейнер удалить и пересоздать.

---

### Сервис `app` — Nextcloud

```yaml
app:
  image: nextcloud:32.0.5   # конкретная версия вместо latest
  restart: always
  ports:
    - "80:80"     # HTTP, доступ в локальной сети
    - "443:443"   # HTTPS, для ZeroTier
  volumes:
    - ~/nextcloud/data:/var/www/html   # файлы Nextcloud сохраняются на сервере
  depends_on:
    - db           # Docker Compose сначала запустит базу, потом приложение
  environment:
    - MYSQL_HOST=db          # имя контейнера — Docker Compose связывает их по имени
    - MYSQL_DATABASE=nextcloud
    - MYSQL_USER=nextcloud
    - MYSQL_PASSWORD=пароль_для_nextcloud
```

---

## Запуск

```bash
cd ~/nextcloud
docker compose up -d
```

Docker Compose прочитает файл, запустит контейнер с MariaDB, затем контейнер с Nextcloud, автоматически создаст между ними сеть и откроет порты. После этого Nextcloud будет доступен в браузере по IP-адресу сервера.

---

## Узкие места и планы по улучшению

- **Непереносимость путей** — в `volumes` используются пути через `~` (домашняя директория `/home/username`), а не относительные через `.`. Если сменится пользователь, контейнер не запустится. Для домашнего сервера с одним пользователем это некритично.

- **Нет разграничения по пользователям** — стоит разобраться с правами.

- **Нет кэширования через Redis** — добавление Redis может ускорить работу Nextcloud на слабом железе Dexp Compact M008.

- **`latest` заменён на конкретную версию** — `nextcloud:32.0.5` вместо `nextcloud:latest`, чтобы избежать неожиданных обновлений.

