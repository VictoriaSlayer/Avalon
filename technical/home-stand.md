## Домашний стенд

> Статус: черновик
> Обновлено: 2026-10-27 23:05 MSK
> Ответственный: Claude

Устройство домашнего стенда: версии, запуск, хранение, сети, проверки. Структура повторяет пункты из «Необходимо описать окружение и версии.md».

**Не проверено:** сведения собраны по записям установки (2026-10-06 – 2026-10-21) и по файлам репозитория Superset тега 3.1.1, а не сняты с компьютера сегодня. Команды для сверки в разделе 10.

**Ограничение:** по правилам папки `technical` здесь нет логинов, паролей, сведений о дисках и личных путей. Вместо них используются заглушки:

| Заглушка | Что означает |
|---|---|
| `<DATA_DIR>` | отдельный раздел Ubuntu для данных BI |
| `<STAND_DIR>` | каталог с конфигурациями стенда в домашней папке |
| `<DB_USER>`, `<DB_PASSWORD>` | роль и пароль учебной базы |

## 1. Версии ПО

| Компонент | Версия |
|---|---|
| Ubuntu | 26.04.1 |
| Docker Engine | 29.8.2 |
| Docker Compose | 5.6.0 (плагин `docker compose`) |
| PostgreSQL (учебная база) | образ `postgres:17` (17.11) |
| Superset | образ `apache/superset:3.1.1` (Docker Hub) |
| Служебная база Superset | образ `postgres:15` |
| Кэш Superset | образ `redis:7` |

Теги `postgres:15` и `redis:7` заданы в `docker-compose-non-dev.yml` тега 3.1.1. Это плавающие теги: точная minor-версия зависит от даты загрузки образа.

**Не проверено:** digest образов и фактические minor-версии `postgres:15` и `redis:7`.

## 2. Конфигурация запуска

Используются два независимых Compose-проекта.

| Сервис | Файл | Происхождение |
|---|---|---|
| Учебный PostgreSQL | `<STAND_DIR>/postgres/compose.yaml` | написан вручную |
| Superset | `<STAND_DIR>/superset/docker-compose-non-dev.yml` | из репозитория Superset (`git clone`, тег 3.1.1), изменена одна строка |

Запуск:

```bash
cd <STAND_DIR>/postgres && docker compose up -d
cd <STAND_DIR>/superset && docker compose -f docker-compose-non-dev.yml up -d
```

Единственное изменение в репозитории Superset — строка образа в `docker-compose-non-dev.yml`. Без неё используется `latest-dev`:

```
было:  x-superset-image: &superset-image apachesuperset.docker.scarf.sh/apache/superset:${TAG:-latest-dev}
стало: x-superset-image: &superset-image apache/superset:3.1.1
```

Первоначальная настройка Superset выполняется одноразовым контейнером `superset_init` (скрипт `docker/docker-init.sh`): миграции служебной базы, создание администратора, инициализация ролей и загрузка примеров (включена `SUPERSET_LOAD_EXAMPLES=yes`). Контейнер запускается при каждом `docker compose up`, повторные шаги на уже настроенной базе не ломают её.

При первом запуске служебной базы скрипт из `docker/docker-entrypoint-initdb.d` создаёт дополнительную базу `examples` для примеров.

Dockerfile собственной сборки нет.

## 3. Переменные окружения

| Где | Переменные | Обязательны | Где используются |
|---|---|---|---|
| `<STAND_DIR>/postgres/.env` | `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | да | контейнер учебного PostgreSQL |
| `<STAND_DIR>/superset/docker/.env-non-dev` | см. таблицу ниже | да | все контейнеры Superset и служебная база |

Пример безопасных значений для учебного PostgreSQL (будущий `infrastructure/.env.example`):

```
POSTGRES_DB=study
POSTGRES_USER=<DB_USER>
POSTGRES_PASSWORD=<DB_PASSWORD>
```

Переменные `.env-non-dev` (файл не менялся, значения из репозитория тега 3.1.1):

| Переменная | Назначение |
|---|---|
| `COMPOSE_PROJECT_NAME=superset` | имя Compose-проекта, задаёт префикс томов и сети |
| `DATABASE_DB`, `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_USER`, `DATABASE_PASSWORD`, `DATABASE_DIALECT` | подключение Superset к служебной базе |
| `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | создание служебной базы контейнером `superset_db` |
| `EXAMPLES_DB`, `EXAMPLES_HOST`, `EXAMPLES_PORT`, `EXAMPLES_USER`, `EXAMPLES_PASSWORD` | база с примерами |
| `REDIS_HOST`, `REDIS_PORT` | подключение к Redis |
| `SUPERSET_SECRET_KEY` | ключ шифрования сессий и паролей подключений |
| `SUPERSET_LOAD_EXAMPLES` | загружать ли примеры при init (`yes`) |
| `SUPERSET_ENV`, `SUPERSET_PORT`, `PYTHONPATH`, `CYPRESS_CONFIG`, `MAPBOX_API_KEY` | служебные настройки |

**Риск:** в `.env-non-dev` стоят публичные значения из репозитория: пароли `superset`/`examples` и ключ `SUPERSET_SECRET_KEY=TEST_NON_DEV_SECRET`. Для домашнего учебного стенда допустимо, на рабочем стенде нужны собственные пароли и ключ.

## 4. Хранение данных

| Сервис | Где хранится на Ubuntu | Путь в контейнере | Сохранять между перезапусками |
|---|---|---|---|
| Учебный PostgreSQL | каталог `<DATA_DIR>/postgres` (bind mount) | `/var/lib/postgresql/data` | да |
| Служебная база Superset (дашборды, графики, пользователи) | Docker volume `superset_db_home` | `/var/lib/postgresql/data` | да |
| Superset (домашний каталог) | Docker volume `superset_superset_home` | `/app/superset_home` | да |
| Redis | Docker volume `superset_redis` | `/data` | желательно, см. ниже |
| Скрипты и настройки Superset | каталог `<STAND_DIR>/superset/docker` (bind mount) | `/app/docker` | это часть репозитория |

Все Docker volumes физически лежат в хранилище Docker, перенесённом на `<DATA_DIR>/docker`.

Прочие каталоги на `<DATA_DIR>`: `datasets` (CSV для загрузки), `backups` (резервные копии `pg_dump`).

Имена томов: в compose заданы `db_home`, `superset_home`, `redis`, Docker добавляет к ним префикс проекта `superset_`. **Не проверено:** фактические имена на машине, проверка `docker volume ls`.

Redis хранит кэш и очередь фоновых задач Celery. Потеря кэша безвредна. Потеря очереди отменяет ещё не выполненные фоновые задачи (отчёты, рассылки); пока их нет, это некритично.

## 5. Сети и порты

| Сервис | Сеть | Опубликованный порт |
|---|---|---|
| Учебный PostgreSQL | сеть своего Compose-проекта | `127.0.0.1:5432` (только этот компьютер) |
| Superset (веб) | сеть Compose-проекта Superset | `8088:8088`, на всех сетевых интерфейсах |
| Служебная база Superset, Redis, workers | сеть Compose-проекта Superset | не публикуются |

**Открытый вопрос:** Superset и учебный PostgreSQL находятся в разных Docker-сетях. Из контейнера Superset адрес `localhost` указывает на сам контейнер, поэтому подключение к `study` нужно настроить отдельно (общая сеть или `host.docker.internal`).

**Риск:** Superset открыт на всех интерфейсах, то есть доступен с других компьютеров сети. Дома это допустимо. На рабочем стенде публиковать `127.0.0.1:8088` и подключаться через SSH-туннель.

## 6. PostgreSQL

| Параметр | Значение |
|---|---|
| База | `study` |
| Роль | `<DB_USER>`, владелец базы (создаётся образом из `POSTGRES_USER`) |
| Схемы | `public` (своих схем пока нет) |
| Расширения | не устанавливались |
| Скрипты инициализации | нет |

**Не проверено:** кодировка и часовой пояс базы. **Предположение:** UTF8 и UTC по умолчанию образа.

## 7. Superset

| Параметр | Значение |
|---|---|
| Пользовательские настройки | нет, `superset_config.py` не менялся |
| Драйвер PostgreSQL | `psycopg2`, входит в образ |
| Дополнительные зависимости | не устанавливались |
| Служебная база | хранит метаданные Superset: подключения, датасеты, графики, дашборды, пользователей |
| Redis | кэш и брокер очереди задач Celery |
| `superset_worker`, `superset_worker_beat` | фоновые и плановые задачи (Celery): отчёты, асинхронные запросы |

## 8. Настройки Ubuntu

| Настройка | Зачем | Только из-за домашней разметки |
|---|---|---|
| Автоподключение раздела данных в `/etc/fstab` с опцией `nofail` | раздел `<DATA_DIR>` | да |
| `/etc/docker/daemon.json`: `data-root` = `<DATA_DIR>/docker` | хранилище Docker на разделе данных | да |
| systemd drop-in `RequiresMountsFor=<DATA_DIR>` для `docker.service` | Docker стартует только после подключения раздела | да |
| Пользователь в группе `docker` | команды Docker без `sudo` | нет |
| Автозапуск Docker (`systemctl enable docker`) | стенд поднимается после перезагрузки | нет |

**Важно:** на домашнем компьютере стоит вторая ОС. Её быстрый запуск отключён, иначе она может повредить раздел данных.

## 9. Проверки

- [x] Docker запускается без `sudo`.
- [x] Учебный PostgreSQL запущен, подключение из DBeaver к `study` работает.
- [x] Интерфейс Superset открывается на порту 8088, вход выполнен.
- [ ] Подключение Superset к PostgreSQL `study`.
- [ ] График на синтетических данных.
- [ ] Работа стенда после перезагрузки компьютера.

**Не проверено:** отметки «выполнено» взяты из записей установки, повторно сегодня не проверялись.

## 10. Как сверить с компьютером

```bash
lsb_release -d
docker --version && docker compose version
docker info | grep "Docker Root Dir"
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
docker images --digests
docker volume ls
docker network ls
cd <STAND_DIR>/superset && git describe --tags && git status --short
```

Ожидаемый результат последней команды: `3.1.1` и строка ` M docker-compose-non-dev.yml`.
