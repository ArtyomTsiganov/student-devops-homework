# Домашнее задание

Продолжите Flask-приложение первой пары в отдельном репозитории: добавьте постоянные заметки в PostgreSQL и счётчик в Redis. Опишите приложение, входной сервис Nginx, БД и кеш в `compose.yaml`, запустите стек на своей Linux-ВМ. Закрепите зависимости приложения, отделите установку зависимостей от копирования кода. Добавьте healthcheck БД и кеша и настройте ожидание app через `depends_on` с `condition: service_healthy`. Подтвердите готовность выводом `docker compose ps` и запросом к приложению.


Поднялось 4 контейнера, с указанными настройками
```
NAME                       IMAGE                  COMMAND                  SERVICE   CREATED         STATUS                   PORTS
homework-compose-app-1     homework-compose-app   "python app.py"          app       4 seconds ago   Up Less than a second    8000/tcp
homework-compose-cache-1   redis:8.8-alpine       "docker-entrypoint.s…"   cache     4 seconds ago   Up 4 seconds (healthy)   6379/tcp
homework-compose-db-1      postgres:18            "docker-entrypoint.s…"   db        4 seconds ago   Up 3 seconds (healthy)   5432/tcp
homework-compose-web-1     nginx:1.30-alpine      "/docker-entrypoint.…"   web       4 seconds ago   Up Less than a second    0.0.0.0:18090->80/tcp, [::]:18090->80/tcp
```

Подключите к БД именованный volume по пути, соответствующему выбранной версии образа. Запишите контрольную заметку, замените контейнер БД с подключением прежнего тома и запросом SQL или через приложение подтвердите её сохранность. Счётчик Redis может быть временным: явно опишите, какое состояние сохраняете, а какое допускаете потерять.

```
◄ 30s ◎ docker volume create --label homework=storage homework-data             □ student-pack/homework 13:45
homework-data

◄ 0s ◎ docker volume inspect homework-data                                      □ student-pack/homework 13:52
[
    {
        "CreatedAt": "2026-10-12T13:52:58+05:00",
        "Driver": "local",
        "Labels": {
            "homework": "storage"
        },
        "Mountpoint": "/var/lib/docker/volumes/homework-data/_data",
        "Name": "homework-data",
        "Options": null,
        "Scope": "local"
    }
]

◄ 0s ◎ docker run -d --name homework-db -e POSTGRES_HOST_AUTH_METHOD=trust -v homework-data:/var/lib/postgresql postgres:18
7ed5b87a6e0366529bbebfc725ee06c4d5faf3cc00bd30cdf08f4940c227574d

◄ 0s ◎ docker ps                                                                □ student-pack/homework 14:02
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS                          PORTS      NAMES
7ed5b87a6e03   postgres:18            "docker-entrypoint.s…"   4 seconds ago    Up 3 seconds                    5432/tcp   homework-db
a153c1a4ed17   nginx:1.30-alpine      "/docker-entrypoint.…"   17 minutes ago   Restarting (1) 55 seconds ago              homework-compose-web-1
d9611afdae71   homework-compose-app   "python app.py"          17 minutes ago   Restarting (1) 18 seconds ago              homework-compose-app-1
046b28cdf7c8   postgres:18            "docker-entrypoint.s…"   17 minutes ago   Up 17 minutes (healthy)         5432/tcp   homework-compose-db-1
8b4a56914628   redis:8.8-alpine       "docker-entrypoint.s…"   17 minutes ago   Up 17 minutes (healthy)         6379/tcp   homework-compose-cache-1

◄ 0s ◎ docker exec homework-db bash -c 'until pg_isready -U postgres >/dev/null 2>&1; do sleep 1; done'

◄ 0s ◎ docker exec homework-db psql -U postgres -c  "CREATE TABLE demo (id int PRIMARY KEY, note text); INSERT INTO demo VALUES (1, 'переживалово');"
CREATE TABLE
INSERT 0 1

◄ 0s ◎ docker rm -f homework-db                                                 □ student-pack/homework 14:04
homework-db

◄ 0s ◎ docker run -d --name homework-db -e POSTGRES_HOST_AUTH_METHOD=trust -v homework-data:/var/lib/postgresql postgres:18
8907a634f0355757d750ba9ce24024e4aa80ff9c6c8a00a559f4c23586612374

◄ 0s ◎ docker exec homework-db psql -U postgres -At -c 'SELECT id, note FROM demo'
1|переживалово

```

Сделайте холодный бэкап БД: остановите все сервисы, которые могут писать в неё, и саму БД, сохранив том обычным `docker compose down` без `-v`. Через временный контейнер создайте архив содержимого тома в каталоге вне исходного volume. После успешного создания архива удалите только том собственного учебного проекта, создайте его заново и восстановите архив. Поднимите тот же стек с тем же образом БД и подтвердите контрольную заметку запросом. В отчёте приложите команды остановки, архивирования, удаления, восстановления и ключевой вывод проверки. Создание архива без проверки восстановления не считается завершённой частью задания.




```
◄ 3s ◎ docker compose exec -T db sh -c 'until pg_isready -U compose_user -d compose_demo >/dev/null 2>&1; do sleep 1; done'
                                                                                                             
◄ 0s ◎ docker compose exec -T db psql -U compose_user -d compose_demo -c "CREATE TABLE backup_check (id integer PRIMARY KEY, note text); INSERT INTO backup_check VALUES (1, 'чупачупс');"
CREATE TABLE
INSERT 0 1
                                                                                                             
◄ 0s ◎ docker compose exec -T db psql -U compose_user -d compose_demo -At -c "SELECT id, note FROM backup_check;"
1|чупачупс

◄ 0s ◎ docker compose down                                                      □ student-pack/homework 14:53
[+] down 6/6
 ✔ Container homework-compose-web-1   Removed                                                            0.2s
 ✔ Container homework-compose-app-1   Removed                                                            0.0s
 ✔ Container homework-compose-db-1    Removed                                                            0.1s
 ✔ Container homework-compose-cache-1 Removed                                                            0.2s
 ✔ Network homework-compose_back      Removed                                                            0.2s
 ✔ Network homework-compose_front     Removed                                                            0.1s
                                                                                                             
◄ 0s ◎ docker run --rm --entrypoint tar \                                       □ student-pack/homework 14:53
             -v homework-compose_pgdata:/data:ro \
             -v "$PWD/backup:/backup" \
             postgres:18 \
             czf /backup/homework-compose_pgdata.tar.gz -C /data .
                                                                                                             
◄ 0s ◎ ls -lh backup/homework-compose_pgdata.tar.gz                             □ student-pack/homework 14:54
       tar -tzf backup/homework-compose_pgdata.tar.gz | head
Permissions Size User Date Modified Name
.rw-r--r--  7,0M root 12 окт 14:54   backup/homework-compose_pgdata.tar.gz
./
./18/
./18/docker/
./18/docker/pg_wal/
./18/docker/pg_wal/archive_status/
./18/docker/pg_wal/summaries/
./18/docker/pg_wal/000000010000000000000001
./18/docker/global/
./18/docker/global/pg_control
./18/docker/global/1262
                                                                                                             
◄ 0s ◎ docker volume rm homework-compose_pgdata                                 □ student-pack/homework 14:54
homework-compose_pgdata
                                                                                                             
◄ 0s ◎ docker volume create homework-compose_pgdata                             □ student-pack/homework 14:54
homework-compose_pgdata
                                                                                                   
◄ 0s ◎ docker run --rm --entrypoint tar \                                       □ student-pack/homework 14:54
             -v homework-compose_pgdata:/data \
             -v "$PWD/backup:/backup:ro" \
             postgres:18 \
             xzf /backup/homework-compose_pgdata.tar.gz -C /data
                                                                                                   
◄ 0s ◎ docker compose up -d                                                     □ student-pack/homework 14:54
WARN[0000] volume "homework-compose_pgdata" already exists but was not created by Docker Compose. Use `external: true` to use an existing volume 
[+] up 6/6
 ✔ Network homework-compose_back      Created                                                            0.0s
 ✔ Network homework-compose_front     Created                                                            0.0s
 ✔ Container homework-compose-cache-1 Healthy                                                            2.4s
 ✔ Container homework-compose-db-1    Healthy                                                            3.4s
 ✔ Container homework-compose-app-1   Started                                                            3.4s
 ✔ Container homework-compose-web-1   Started                                                            3.5s
                                                                                                   
◄ 3s ◎ docker compose exec -T db sh -c 'until pg_isready -U compose_user -d compose_demo >/dev/null 2>&1; do sleep 1; done'

◄ 0s ◎ docker compose exec -T db psql -U compose_user -d compose_demo -At -c "SELECT id, note FROM backup_check;"
1|чупачупс

```
Разделите стек минимум на две пользовательские сети: front для Nginx и Flask, back для Flask, БД и кеша. Публикуйте только входной сервис. Из Nginx проверьте отсутствие DNS-имени БД и отказ прямого TCP-подключения к её внутреннему IP и порту; из приложения подтвердите успешный доступ к БД и кешу. Не подключайте входной сервис к back ради прохождения проверки. Объясните, почему Flask в двух сетях выполняет свои запросы к БД, но не становится автоматическим IP-маршрутизатором.


```
◄ 5s ◎ docker compose exec app getent hosts db cache                            □ student-pack/homework 15:07
172.20.0.2      db
172.20.0.3      cache
                                                                                                             
◄ 0s ◎ docker compose exec web sh -c 'getent hosts db || echo "db: имя не найдено из web"'
db: имя не найдено из web
                                                                                                             
◄ 7s ◎ docker compose ps --format 'table {{.Name}}\t{{.Service}}'               □ student-pack/homework 15:08
NAME                       SERVICE
homework-compose-app-1     app
homework-compose-cache-1   cache
homework-compose-db-1      db
homework-compose-web-1     web
                                                                                                             
◄ 0s ◎ docker network ls --filter name=^home                                    □ student-pack/homework 15:08
NETWORK ID     NAME                     DRIVER    SCOPE
40194dd023bc   homework-compose_back    bridge    local
343d7a123359   homework-compose_front   bridge    local
                                                                                                             
◄ 0s ◎ docker volume ls --filter name=^home                                     □ student-pack/homework 15:08
DRIVER    VOLUME NAME
local     homework-compose_pgdata
local     homework-data


curl -fsS --retry 20 --retry-delay 1 --retry-connrefused localhost:18090/□ student-pack/homework 15:08
       curl -fsS localhost:18090/
       curl -fsS --get --data-urlencode 'text=hello from compose' localhost:18090/notes/add
       curl -fsS localhost:18090/notes
Docker Compose demo - visits: 1
Docker Compose demo - visits: 2
added: hello from compose
1: hello from compose
                        
◄ 0s ◎ docker compose exec web sh -c "if nc -z -w 2 172.20.0.2 5432; then echo 'ОШИБКА: TCP-доступ к БД есть'
; exit 1; else echo 'TCP-доступ из web отсутствует'; fi"
TCP-доступ из web отсутствует
                                 


Наличие у Flask двух сетевых интерфейсов само по себе не позволяет Nginx обращаться к БД напрямую. Docker изолирует пользовательские сети, а Flask выполняет запросы к БД и Redis от своего имени.
```

Вынесите изменяемые значения в переменные. Добавьте в Git `.env.example` с безопасными примерными значениями и исключите рабочую `.env` через `.gitignore`. Покажите итог `docker compose config`, удалив реальные секреты из отчётного вывода. Объясните источник имени проекта, его приоритет и имена контейнеров, сетей и томов. Покажите различие `down` и `down -v` только на данных собственного стенда, которые можно восстановить.


Имя проекта - homework-compose. Источник - поле name: в compose.yaml


Я загуглил и приоритет источников имени проекта такой:
1. Параметр -p, явно переданный команде.
2. Переменная окружения COMPOSE_PROJECT_NAME.
3. Поле name: в файле Compose.
4. Имя каталога проекта, если остальные источники отсутствуют.


```
name: homework-compose
services:
  app:
    restart: unless-stopped
    environment:
      DB_HOST: db
      POSTGRES_DB: compose_demo
      POSTGRES_PASSWORD: "[REDACTED]"
      POSTGRES_USER: compose_user
      REDIS_HOST: cache
    mem_limit: "268435456"
    networks:
      back: null
      front: null
    build:
      context: /home/tsiga/Документы/docker-exp/student-pack/homework
      dockerfile: Dockerfile
    depends_on:
      cache:
        condition: service_healthy
        required: true
      db:
        condition: service_healthy
        required: true
  cache:
    restart: unless-stopped
    image: redis:8.8-alpine
    networks:
      back: null
    healthcheck:
      test:
        - CMD
        - redis-cli
        - ping
      timeout: 2s
      interval: 2s
      retries: 15
  db:
    restart: unless-stopped
    environment:
      POSTGRES_DB: compose_demo
      POSTGRES_PASSWORD: "[REDACTED]"
      POSTGRES_USER: compose_user
    image: postgres:18
    networks:
      back: null
    volumes:
      - type: volume
        source: pgdata
        target: /var/lib/postgresql
        volume: {}
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB
      timeout: 2s
      interval: 2s
      retries: 15
      start_period: 3s
  web:
    restart: unless-stopped
    image: nginx:1.30-alpine
    networks:
      front: null
    volumes:
      - type: bind
        source: /home/tsiga/Документы/docker-exp/student-pack/homework/nginx/default.conf
        target: /etc/nginx/conf.d/default.conf
        read_only: true
        bind: {}
    depends_on:
      app:
        condition: service_started
        required: true
    ports:
      - mode: ingress
        target: 80
        published: "18090"
        protocol: tcp
networks:
  back:
    name: homework-compose_back
  front:
    name: homework-compose_front
volumes:
  pgdata:
    name: homework-compose_pgdata

```

Со звёздочкой: добавьте `compose.override.yaml` или `develop.watch`, чтобы изменение кода применялось автоматически. Объясните базовую и объединённую конфигурацию, а для watch покажите фактическое событие sync/rebuild, не только запуск наблюдения. Ещё одно независимое задание со звёздочкой: добавьте в GitHub Actions `CI=true dive` с осмысленными порогами. Проверка должна возвращать ошибку job при нарушении порога; демонстрационный `|| true` для этого не подходит.

Сдайте репозиторий с исходниками, Dockerfile, compose-файлами и `.env.example`, а также короткий отчёт с командами, контрольными строками и объяснениями. Не присылайте токены, пароли рабочей среды, `.env` или полный дамп личных данных. В конце очистите только ресурсы своего проекта.
