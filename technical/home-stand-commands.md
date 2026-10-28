## Домашний стенд: запуск, остановка, обслуживание

> Статус: черновик
> Обновлено: 2026-10-28 06:20 MSK
> Ответственный: Claude

Справочник команд для домашнего стенда: где лежит каждая программа, как её запустить, остановить и что с ней можно делать. Устройство стенда (версии, тома, сети) описано в `home-stand.md`.

**Ограничение:** по правилам папки `technical` вместо личных путей и логинов стоят заглушки. Подставляй свои значения при вводе команд.

| Заглушка | Что подставить |
|---|---|
| `<STAND_DIR>` | каталог стенда в домашней папке (в нём лежат папки `postgres` и `superset`) |
| `<DATA_DIR>` | точка подключения раздела данных |
| `<DB_USER>` | роль учебной базы из `<STAND_DIR>/postgres/.env` (`POSTGRES_USER` в `compose.yaml`) |

**Не проверено:** команды собраны по записям установки и гайдам, повторно в этом виде не выполнялись.

## 1. Что где находится

| Программа | Где настройки | Как работает | Адрес |
|---|---|---|---|
| Docker | служба Ubuntu `docker`, хранилище на `<DATA_DIR>/docker` | запускается сам при включении компьютера | — |
| Учебный PostgreSQL | `<STAND_DIR>/postgres/` (`compose.yaml`, `.env`) | 1 контейнер `bi-postgres` | `localhost:5432`, база `study` |
| Superset | `<STAND_DIR>/superset/` (`docker-compose-non-dev.yml`, `docker/.env-non-dev`) | 6 контейнеров: `superset_app`, `superset_db`, `superset_cache`, `superset_worker`, `superset_worker_beat`, `superset_init` | http://localhost:8088 |
| DBeaver | обычное приложение (snap) | запускается вручную | подключение к `localhost:5432` |

**Важно:** команды `docker compose` работают с тем проектом, в каталоге которого ты находишься. Поэтому перед ними всегда `cd` в нужный каталог.

**Важно:** у Superset в каталоге лежит ещё файл для разработчиков `docker-compose.yml`. Поэтому все команды Superset пишутся с `-f docker-compose-non-dev.yml`. Без этой части команда обратится не к твоему стенду.

## 2. Всё сразу

### После включения компьютера

Ничего делать не нужно: раздел данных подключается сам, затем стартует Docker, затем контейнеры (у всех политика `restart: unless-stopped`). Проверка:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

Ожидаемо: `bi-postgres` и пять контейнеров `superset_*` со статусом `Up`. `superset_init` в списке нет, это нормально: он отработал и выключился.

**Важно:** контейнер, остановленный вручную командой `stop`, после перезагрузки сам **не** поднимется. Его нужно запустить командой `start`.

### Остановить весь стенд (например, чтобы освободить память)

```bash
cd <STAND_DIR>/superset && docker compose -f docker-compose-non-dev.yml stop
cd <STAND_DIR>/postgres && docker compose stop
```

Сначала Superset, потом база. Данные сохраняются.

### Запустить весь стенд после остановки

```bash
cd <STAND_DIR>/postgres && docker compose start
cd <STAND_DIR>/superset && docker compose -f docker-compose-non-dev.yml start
```

Сначала база, потом Superset. Интерфейс Superset открывается через 1–2 минуты.

## 3. Docker

| Действие | Команда | Пояснение |
|---|---|---|
| Состояние службы | `systemctl status docker` | ищи `active (running)`; выход `q` |
| Запустить службу | `sudo systemctl start docker` | если она остановлена |
| Проверить автозапуск | `systemctl is-enabled docker` | ожидаемо `enabled` |
| Все контейнеры, включая остановленные | `docker ps -a` | |
| Сколько памяти и процессора едят контейнеры | `docker stats` | выход `Ctrl+C` |
| Сколько места занимает Docker | `docker system df` | |
| Свободное место на разделе данных | `df -h <DATA_DIR>` | |

**Важно:** Docker стартует только после подключения раздела `<DATA_DIR>`. Если Docker не запустился, сначала проверь раздел: `findmnt <DATA_DIR>`. Пустой ответ значит, что раздел не подключён (см. гайд восстановления раздела данных).

## 4. Учебный PostgreSQL

Все команды из каталога проекта:

```bash
cd <STAND_DIR>/postgres
```

| Действие | Команда | Данные |
|---|---|---|
| Состояние | `docker compose ps` | — |
| Остановить | `docker compose stop` | сохраняются |
| Запустить после остановки | `docker compose start` | — |
| Перезапустить | `docker compose restart` | сохраняются |
| Создать или пересоздать контейнер (после правки `compose.yaml` или если контейнера нет) | `docker compose up -d` | сохраняются |
| Удалить контейнер | `docker compose down` | сохраняются на `<DATA_DIR>/postgres` |
| Журнал, последние 50 строк | `docker compose logs --tail 50` | — |
| Журнал в реальном времени | `docker compose logs -f` | выход `Ctrl+C` |

### Что можно делать с базой

**Открыть консоль SQL в терминале:**

```bash
docker exec -it bi-postgres psql -U <DB_USER> -d study
```

Полезные команды внутри: `\dt` (список таблиц), `\d имя_таблицы` (структура таблицы), `\q` (выход). Каждый SQL-запрос заканчивается `;`.

**Работать мышкой:** DBeaver, подключение `localhost`, порт `5432`, база `study`, пользователь `<DB_USER>`, пароль из `<STAND_DIR>/postgres/.env`.

**Сделать резервную копию:**

```bash
docker exec bi-postgres pg_dump -U <DB_USER> -Fc study > <DATA_DIR>/backups/study_$(date +%F).dump
```

Создаёт файл вида `study_2026-10-28.dump`. Проверить: `ls -lh <DATA_DIR>/backups/`. Размер должен быть больше нуля.

**Не проверено:** восстановление из копии. Его порядок будет описан отдельно.

## 5. Superset

Все команды из каталога проекта:

```bash
cd <STAND_DIR>/superset
```

| Действие | Команда | Данные |
|---|---|---|
| Состояние | `docker compose -f docker-compose-non-dev.yml ps` | — |
| Остановить | `docker compose -f docker-compose-non-dev.yml stop` | сохраняются |
| Запустить после остановки | `docker compose -f docker-compose-non-dev.yml start` | — |
| Перезапустить всё | `docker compose -f docker-compose-non-dev.yml restart` | сохраняются |
| Перезапустить только веб-интерфейс | `docker compose -f docker-compose-non-dev.yml restart superset` | сохраняются |
| Создать или пересоздать контейнеры | `docker compose -f docker-compose-non-dev.yml up -d` | сохраняются; заново отработает `superset_init` |
| Удалить контейнеры | `docker compose -f docker-compose-non-dev.yml down` | сохраняются в томах Docker |
| Журнал веб-интерфейса | `docker logs --tail 50 superset_app` | — |
| Журнал инициализации | `docker logs superset_init` | — |
| Журнал фоновых задач | `docker logs --tail 50 superset_worker` | — |

В командах `restart`, `logs` и других используются имена **сервисов** из compose-файла: `superset`, `db`, `redis`, `superset-worker`, `superset-worker-beat`, `superset-init`. Команда `docker logs` берёт имена **контейнеров**: `superset_app`, `superset_db` и т. д.

### Что можно делать

**Открыть:** http://localhost:8088, логин `admin`.

**Сменить пароль `admin`:** в интерфейсе меню пользователя справа вверху → Info → Reset my password. Или в терминале (пароль вводится скрыто):

```bash
docker exec -it superset_app superset fab reset-password --username admin
```

**Проверить, что веб-интерфейс жив:**

```bash
curl -s http://localhost:8088/health
```

Ожидаемо: `OK`.

**Резервная копия метаданных (подключения, датасеты, графики, дашборды, пользователи):**

```bash
docker exec superset_db pg_dump -U superset -Fc superset > <DATA_DIR>/backups/superset_meta_$(date +%F).dump
```

**Открытый вопрос:** Superset пока не подключён к базе `study`, это отдельный шаг.

**Риск:** порт 8088 открыт для всей домашней сети, не только для этого компьютера. Поэтому стандартный пароль `admin` лучше сменить.

## 6. DBeaver

| Действие | Как |
|---|---|
| Запустить | меню приложений → DBeaver, или в терминале `dbeaver-ce &` |
| Остановить | закрыть окно |
| Обновить | `sudo snap refresh dbeaver-ce` |

DBeaver работает, только если запущен `bi-postgres`.

## 7. Опасные команды

Не выполнять, если не понимаешь последствий. Ни одна из них не нужна для обычной работы.

| Команда | Что удалит |
|---|---|
| `docker compose -f docker-compose-non-dev.yml down -v` (в каталоге Superset) | все дашборды, графики, подключения и пользователей Superset |
| `docker volume rm …`, `docker volume prune` | данные томов Docker |
| `docker system prune -a --volumes` | все неиспользуемые образы и тома |
| `sudo rm -rf <DATA_DIR>/postgres` | учебную базу `study` |
| Перезапуск Windows с включённым «Быстрым запуском» | может повредить раздел данных |

Команда `docker compose down` без `-v` безопасна: удаляет контейнеры, данные остаются.

## 8. Если что-то не работает

| Симптом | Что проверить |
|---|---|
| `docker ps` пишет `permission denied` | перелогинься; проверь, что пользователь в группе docker: `groups` |
| `Cannot connect to the Docker daemon` | `systemctl status docker`, затем `findmnt <DATA_DIR>` |
| `bi-postgres` нет в `docker ps` | `docker ps -a`: если `Exited`, смотри `docker compose logs --tail 50` в каталоге postgres, затем `docker compose up -d` |
| DBeaver: `Connection refused` | запущен ли `bi-postgres` |
| DBeaver: `password authentication failed` | пароль в `<STAND_DIR>/postgres/.env` (открой файл в редакторе, в чат не копируй) |
| http://localhost:8088 не открывается | `docker compose -f docker-compose-non-dev.yml ps` в каталоге Superset; после запуска подожди 1–2 минуты |
| `no configuration file provided` или незнакомые контейнеры | команда выполнена не в том каталоге или без `-f docker-compose-non-dev.yml` |
| `port is already allocated` | порт занят другой программой: `sudo ss -ltnp \| grep -E '5432\|8088'` |

Журналы и `.env` в чат не копируй целиком: в них могут быть пароли. Присылай только строки с ошибкой.
