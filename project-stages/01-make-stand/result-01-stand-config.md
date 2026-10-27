## Результат 01: конфигурация BI-стенда для развёртывания на рабочей ВМ

> Статус: черновик, комплект подготовлен, на рабочей ВМ не проверялся
> Обновлено: 2026-10-27 23:30 MSK
> Ответственный: Claude — подготовка; Saint — согласование и выполнение команд; Codex — развёртывание и приёмка

Ответ на `project-stages/01-make-stand/task-01-make-stand.md`. Документ самодостаточен: по нему стенд разворачивается без доступа к домашней машине и без истории чата. Устройство домашнего стенда описано в `technical/home-stand.md`.

**Важно:** «комплект подготовлен» не означает «переносимость подтверждена». Подтверждение — успешное прохождение раздела 8 на рабочей ВМ.

Обозначения статусов проверки: **выполнено** (проверено на машине), **по записям** (из журнала установки дома), **по исходникам** (из файлов Superset тега 3.1.1 на GitHub), **не проверено**.

## 1. Состав стенда

Два независимых Compose-проекта, соединённых общей внешней сетью `bi-net`.

| Проект | Сервис / контейнер | Образ | Назначение |
|---|---|---|---|
| `postgres` | `postgres` / `bi-postgres` | `postgres:17` | учебная база `study` (сюда грузятся CSV) |
| `superset` | `superset` / `superset_app` | `apache/superset:3.1.1` | веб-интерфейс, порт 8088 |
| | `superset-init` / `superset_init` | то же | одноразовая инициализация при каждом `up` |
| | `superset-worker` / `superset_worker` | то же | фоновые задачи Celery |
| | `superset-worker-beat` / `superset_worker_beat` | то же | планировщик Celery |
| | `db` / `superset_db` | `postgres:15` | служебная база метаданных Superset (не учебная) |
| | `redis` / `superset_cache` | `redis:7` | кэш и брокер очереди Celery |

Собственной сборки образов нет. Superset берётся из официального репозитория по тегу `3.1.1`, поверх него кладётся один override-файл (раздел 4.3).

Версии домашнего стенда (по записям, 2026-10-06…13): Ubuntu 26.04.1, Docker Engine 29.8.2, Compose 5.6.0, PostgreSQL 17.11. Теги `postgres:17`, `postgres:15`, `redis:7` плавающие: minor-версия зависит от даты загрузки. **Не проверено:** digest образов домашнего стенда.

## 2. Отличия рабочего стенда от домашнего

| Что | Дома | На рабочей ВМ | Почему |
|---|---|---|---|
| Хранение учебной БД | bind mount на отдельный раздел | named volume `study_data` | отдельного раздела нет и не нужен |
| Docker data-root, fstab, systemd drop-in | перенесены на раздел данных | стандартные | это домашняя особенность (dual boot) |
| Порт 8088 | на всех интерфейсах | `127.0.0.1:8088` + SSH-туннель | ВМ в корпоративной сети |
| Пароли и ключ Superset | публичные значения из репозитория | свои, сгенерированные | обязательная мера безопасности |
| Примеры Superset | загружаются | `SUPERSET_LOAD_EXAMPLES=no` | экономит 5–15 минут и место |
| Маршрут Superset → `study` | не настроен | сеть `bi-net` | дома это недоделка |
| Роль для Superset | владелец базы | отдельная роль только для чтения | разделение прав |

**Открытый вопрос:** домашний стенд пока не переведён на сеть `bi-net`. Решение, переводить ли его, за пользователем; на развёртывание на работе это не влияет.

## 3. Требования и предварительные проверки (на рабочей ВМ)

По диагностике пользователя от 2026-10-23: Ubuntu 26.04.1 x86_64, 10 CPU, ~30 GiB RAM, ~211 GiB свободно, есть sudo, Docker не установлен, HTTPS до Ubuntu, Docker и GitHub доступен. Минимум для стенда: 4 GiB RAM, 15 GiB диска.

Установить Docker Engine и плагин Compose по официальной инструкции для Ubuntu: https://docs.docker.com/engine/install/ubuntu/ (способ «apt repository»). **Не проверено:** поддержка Ubuntu 26.04 в этой инструкции на дату развёртывания, Codex проверяет её перед командами. Также нужны `git` и `openssl` (`sudo apt install -y git openssl`).

Работа с Docker без sudo: `sudo usermod -aG docker $USER`, затем перелогиниться. **Риск:** группа `docker` фактически равна правам root. Если это неприемлемо по политике компании, все команды `docker` выполнять через `sudo`. `chmod 777` не применять.

Проверка (ожидаемо: версии без ошибок, `hello-world` печатает приветствие):

```bash
docker --version && docker compose version
docker run --rm hello-world
```

Нужна версия Compose не ниже 2.24.4 (используется тег `!override`, раздел 4.3).

## 4. Файлы комплекта

Каталог стенда на ВМ далее `<STAND_DIR>` (например `~/bi-stand`). Все файлы `.env` с секретами создаёт пользователь локально, права `600`, в Git они не попадают.

### 4.1. Общая сеть

Выполнить один раз, до запуска проектов:

```bash
docker network create bi-net
```

Ожидаемо: печатается ID сети. Проверка: `docker network ls | grep bi-net`.

### 4.2. Учебный PostgreSQL: `<STAND_DIR>/postgres/`

`compose.yaml` (в Git: `infrastructure/postgres/compose.yaml`):

```yaml
services:
  postgres:
    image: postgres:17
    container_name: bi-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:?}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?}
      POSTGRES_DB: ${POSTGRES_DB:?}
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - study_data:/var/lib/postgresql/data
    networks:
      - bi-net

volumes:
  study_data:

networks:
  bi-net:
    external: true
```

`.env.example` (в Git: `infrastructure/postgres/.env.example`; на ВМ копируется в `.env`):

```
# Административная роль учебной базы. Уникально для окружения.
POSTGRES_USER=<DB_ADMIN_USER>
# Пароль: сгенерировать openssl rand -hex 24. Уникально для окружения.
POSTGRES_PASSWORD=<DB_ADMIN_PASSWORD>
POSTGRES_DB=study
```

Отличия от домашнего файла: логин вынесен в `.env`, bind mount заменён на named volume, добавлена сеть `bi-net`. Путь `/var/lib/postgresql/data` верен для `postgres:17`; при переходе на 18+ путь в образе меняется — это отдельная задача.

### 4.3. Superset: `<STAND_DIR>/superset/`

Получить исходники зафиксированной версии:

```bash
cd <STAND_DIR>
git clone --depth 1 --branch 3.1.1 https://github.com/apache/superset.git
cd superset
git status
```

Ожидаемо: `HEAD detached at 3.1.1`.

Единственная правка исходников — строка образа (иначе берётся `latest-dev`):

```bash
sed -i 's|^x-superset-image: .*|x-superset-image: \&superset-image apache/superset:3.1.1|' docker-compose-non-dev.yml
grep x-superset-image: docker-compose-non-dev.yml
```

Ожидаемо: `x-superset-image: &superset-image apache/superset:3.1.1`.

Положить рядом файл `compose.avalon.yaml` (в Git: `infrastructure/superset/compose.avalon.yaml`):

```yaml
# Поверх docker-compose-non-dev.yml тега 3.1.1.
# Значения environment имеют приоритет над docker/.env-non-dev.
x-avalon-env: &avalon-env
  DATABASE_PASSWORD: ${SUPERSET_DB_PASSWORD:?}
  POSTGRES_PASSWORD: ${SUPERSET_DB_PASSWORD:?}
  EXAMPLES_PASSWORD: ${SUPERSET_EXAMPLES_PASSWORD:?}
  SUPERSET_SECRET_KEY: ${SUPERSET_SECRET_KEY:?}
  SUPERSET_LOAD_EXAMPLES: "no"

services:
  db:
    environment: *avalon-env
  superset:
    environment: *avalon-env
    ports: !override
      - "127.0.0.1:8088:8088"
    networks: [default, bi-net]
  superset-init:
    environment: *avalon-env
  superset-worker:
    environment: *avalon-env
    networks: [default, bi-net]
  superset-worker-beat:
    environment: *avalon-env
    networks: [default, bi-net]

networks:
  bi-net:
    external: true
```

`.env.example` (в Git: `infrastructure/superset/.env.example`; на ВМ копируется в `<STAND_DIR>/superset/.env`, этот путь уже в `.gitignore` репозитория Superset):

```
# Какие compose-файлы применять; позволяет запускать просто docker compose up -d
COMPOSE_FILE=docker-compose-non-dev.yml:compose.avalon.yaml
COMPOSE_PROJECT_NAME=superset
# Пароль служебной базы Superset: openssl rand -hex 24. Уникально для окружения.
SUPERSET_DB_PASSWORD=<SUPERSET_DB_PASSWORD>
# Пароль базы examples (примеры выключены, но база создаётся): openssl rand -hex 24
SUPERSET_EXAMPLES_PASSWORD=<SUPERSET_EXAMPLES_PASSWORD>
# Ключ шифрования сессий и паролей подключений: openssl rand -base64 42. Уникально для окружения.
SUPERSET_SECRET_KEY=<SUPERSET_SECRET_KEY>
```

**Важно:** `SUPERSET_SECRET_KEY` задать до первого запуска и не менять: им шифруются пароли подключений к БД в служебной базе. Потеря ключа = подключения придётся создавать заново. Хранить ключ вне Git, отдельно от резервных копий базы.

**Важно:** пароль служебной базы применяется только при первом создании тома `superset_db_home`. Смена пароля в `.env` позже не меняет пароль в уже созданной базе.

### 4.4. Переменные `docker/.env-non-dev` (файл не меняется)

По исходникам тега 3.1.1. Переменные с секретами переопределяются в `compose.avalon.yaml`.

| Переменная | Потребитель | Значение в файле | На рабочей ВМ |
|---|---|---|---|
| `DATABASE_DB`, `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_USER`, `DATABASE_DIALECT` | Superset → служебная БД | `superset`, `db`, `5432`, `superset`, `postgresql` | без изменений |
| `DATABASE_PASSWORD` | Superset | `superset` | переопределён |
| `POSTGRES_DB`, `POSTGRES_USER` | `superset_db` | `superset` | без изменений |
| `POSTGRES_PASSWORD` | `superset_db` | `superset` | переопределён |
| `EXAMPLES_DB`, `EXAMPLES_HOST`, `EXAMPLES_PORT`, `EXAMPLES_USER` | Superset, скрипт init `superset_db` | `examples`, `db`, `5432`, `examples` | без изменений |
| `EXAMPLES_PASSWORD` | то же | `examples` | переопределён |
| `REDIS_HOST`, `REDIS_PORT` | Superset, Celery | `redis`, `6379` | без изменений |
| `SUPERSET_SECRET_KEY` | Superset | `TEST_NON_DEV_SECRET` | переопределён |
| `SUPERSET_LOAD_EXAMPLES` | `superset_init` | `yes` | `no` |
| `SUPERSET_ENV`, `SUPERSET_PORT`, `PYTHONPATH`, `CYPRESS_CONFIG`, `MAPBOX_API_KEY`, `COMPOSE_PROJECT_NAME` | служебные | — | без изменений |

`superset_config.py` берётся из `docker/pythonpath_dev/` репозитория без изменений: собирает строку подключения из `DATABASE_*`, кэш и Celery на Redis (БД 0 — брокер, БД 1 — результаты и кэш), включён флаг `ALERT_REPORTS`.

## 5. Хранилища

| Хранилище | Тип | Путь в контейнере | Что в нём | Критично для восстановления |
|---|---|---|---|---|
| `postgres_study_data` | named volume | `/var/lib/postgresql/data` | учебная база `study` | да (или повторная загрузка CSV) |
| `superset_db_home` | named volume | `/var/lib/postgresql/data` | метаданные Superset: подключения, датасеты, графики, дашборды, пользователи | да |
| `superset_superset_home` | named volume | `/app/superset_home` | результаты SQL Lab, служебные файлы | нет |
| `superset_redis` | named volume | `/data` | кэш и очередь Celery | нет: потеря кэша безвредна, потеря очереди отменяет невыполненные фоновые задачи |
| `<STAND_DIR>/superset/docker` | bind mount, только чтение по смыслу | `/app/docker` | скрипты запуска и конфигурация из репозитория | восстанавливается из Git |

Имена томов = `<имя проекта>_<имя тома>`. Имя проекта у Superset задано явно (`superset` в `.env`), у учебной базы берётся по имени каталога (`postgres`), поэтому переименование этого каталога изменит имя тома учебной базы. Bind-каталогов с данными нет, настраивать UID/GID не нужно.

Тома переживают `stop`, `down`, пересоздание контейнеров и перезагрузку. Удаляются только `docker compose down -v` и `docker volume rm` (раздел 10).

## 6. Сети и доступ

| Сервис | Сети | Имя в сети | Порт внутри | Опубликован |
|---|---|---|---|---|
| `bi-postgres` | `bi-net` | `bi-postgres` | 5432 | `127.0.0.1:5432` |
| `superset_app` | `superset_default`, `bi-net` | `superset` | 8088 | `127.0.0.1:8088` |
| `superset_worker`, `superset_worker_beat` | `superset_default`, `bi-net` | — | — | нет |
| `superset_db`, `superset_cache`, `superset_init` | `superset_default` | `db`, `redis` | 5432, 6379 | нет |

Из контейнера Superset учебная база доступна по адресу `bi-postgres:5432`. `localhost` внутри контейнера указывает на сам контейнер, а `host.docker.internal` не подходит, так как БД опубликована только на loopback хоста.

Доступ к Superset с рабочего компьютера — через SSH-туннель (на рабочем компьютере):

```bash
ssh -L 8088:127.0.0.1:8088 <VM_USER>@<VM_HOST>
```

После этого в браузере рабочего компьютера: `http://localhost:8088`. Открытие порта 8088 в корпоративную сеть — отдельное решение, не входит в комплект.

## 7. Развёртывание: первый запуск

Все команды на рабочей ВМ в терминале, если не сказано иное.

1. Сеть: раздел 4.1.
2. Учебный PostgreSQL:
   ```bash
   mkdir -p <STAND_DIR>/postgres && cd <STAND_DIR>/postgres
   # положить compose.yaml и .env (из .env.example, со своими значениями)
   chmod 600 .env
   docker compose up -d
   docker compose ps
   ```
   Ожидаемо: `bi-postgres` в статусе `Up`.
3. Роль только для чтения для Superset (пользователь подставляет свой пароль; команда выполняется интерактивно, пароль не попадает в журналы):
   ```bash
   docker exec -it bi-postgres psql -U <DB_ADMIN_USER> -d study
   ```
   ```sql
   CREATE ROLE superset_ro LOGIN PASSWORD '<SUPERSET_RO_PASSWORD>';
   GRANT CONNECT ON DATABASE study TO superset_ro;
   GRANT USAGE ON SCHEMA public TO superset_ro;
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO superset_ro;
   ALTER DEFAULT PRIVILEGES FOR ROLE <DB_ADMIN_USER> IN SCHEMA public GRANT SELECT ON TABLES TO superset_ro;
   \q
   ```
   `<DB_ADMIN_USER>` — суперпользователь кластера (так образ создаёт роль из `POSTGRES_USER`), им грузятся данные. `ALTER DEFAULT PRIVILEGES` даёт чтение таблиц, которые этот пользователь создаст позже.
4. Superset: клонировать и поправить образ (раздел 4.3), положить `compose.avalon.yaml` и `.env`, затем:
   ```bash
   cd <STAND_DIR>/superset
   chmod 600 .env
   docker compose config --quiet && echo OK
   docker compose up -d
   docker logs -f superset_init
   ```
   `config --quiet` проверяет файлы, ничего не печатая (секреты не выводятся). Ждать в журнале `Init Step 3/4 [Complete]`; шага 4 не будет, так как примеры выключены. Выход из журнала `Ctrl+C`.
5. Проверка: `docker compose ps` — `superset_app`, `superset_db`, `superset_cache`, `superset_worker`, `superset_worker_beat` в `Up`, у `superset_app` через 1–2 минуты `healthy`. `superset_init` завершён: `Exited (0)` в выводе `docker compose ps -a`.
6. Вход через туннель (раздел 6): логин `admin`, пароль `admin` — они зашиты в `docker/docker-init.sh` тега 3.1.1. **Сразу сменить пароль:** меню пользователя справа вверху → Info → Reset my password.

## 8. Проверки после развёртывания

1. **Подключение Superset к учебной БД.** Settings → Database Connections → + Database → PostgreSQL, строка подключения:
   ```
   postgresql://superset_ro:<SUPERSET_RO_PASSWORD>@bi-postgres:5432/study
   ```
   Test Connection → «Connection looks good». Драйвер `psycopg2` входит в образ `apache/superset:3.1.1` (по исходникам; дома служебная база работает через него, но подключение к `study` дома не проверялось).
2. **Синтетическая таблица и график.** Под административной ролью:
   ```sql
   CREATE TABLE contracts_demo (id int, signed_on date, status text, amount numeric);
   INSERT INTO contracts_demo VALUES
     (1,'2026-01-15','Согласован',100000),(2,'2026-02-03','На согласовании',250000),
     (3,'2026-02-20','Согласован',75000),(4,'2026-03-11','Отклонён',40000);
   ```
   В Superset: Datasets → + Dataset → `study` / `public` / `contracts_demo`, построить Bar Chart: ось X `status`, метрика `SUM(amount)`. Ожидаемо: три столбца.
3. **Сохранность.** `docker compose stop` и `docker compose start` в обоих каталогах, затем график открывается, таблица на месте.
4. **Перезагрузка ВМ.** После `sudo reboot`: `docker ps` показывает все сервисы `Up` (политика `unless-stopped`, Docker включён в автозапуск: `systemctl is-enabled docker` → `enabled`).

## 9. Последующие запуски и обслуживание

| Действие | Команда (из каталога проекта) |
|---|---|
| Остановить, данные сохраняются | `docker compose stop` |
| Запустить после остановки | `docker compose start` |
| Пересоздать контейнеры после правки compose | `docker compose up -d` |
| Состояние | `docker compose ps` |
| Журнал Superset | `docker logs --tail 50 superset_app` |

При каждом `docker compose up -d` заново выполняется `superset_init`: миграции (без изменений, если их нет), попытка создать admin (если уже есть — пропускается), `superset init`. Данные это не затирает.

Порядок после перезагрузки: ничего делать не нужно; если сеть `bi-net` удалена, контейнеры не стартуют — пересоздать её (раздел 4.1) и выполнить `docker compose up -d` в обоих каталогах.

**Резервное копирование (минимум, не проверено):**

```bash
docker exec bi-postgres pg_dump -U <DB_ADMIN_USER> -Fc study > study_$(date +%F).dump
docker exec superset_db pg_dump -U superset -Fc superset > superset_meta_$(date +%F).dump
```

Плюс отдельно хранить `.env` обоих проектов (без ключа `SUPERSET_SECRET_KEY` восстановленные подключения не расшифруются). Копии агенту не передавать. **Не проверено:** восстановление из этих копий.

## 10. Опасные команды (не входят в обычный сценарий)

| Команда | Последствие |
|---|---|
| `docker compose down -v` | удаляет тома проекта: учебную базу или все метаданные Superset |
| `docker volume rm …`, `docker volume prune` | удаляет данные томов |
| `docker system prune -a --volumes` | удаляет все неиспользуемые образы и тома всех проектов |
| `docker network rm bi-net` | Superset теряет связь с учебной базой, контейнеры не стартуют |
| смена `SUPERSET_SECRET_KEY` после запуска | сохранённые пароли подключений перестают расшифровываться |

## 11. Типовые ошибки и безопасная диагностика

| Симптом | Причина | Проверка |
|---|---|---|
| `network bi-net declared as external, but could not be found` | не создана сеть | `docker network ls` |
| `required variable … is missing` | нет `.env` или пустая переменная | `ls -l .env`; содержимое не выводить |
| `superset_init` падает с ошибкой аутентификации к `db` | пароль в `.env` изменён после создания тома | `docker logs --tail 30 superset_init` |
| Test Connection: `could not translate host name "bi-postgres"` | Superset не подключён к `bi-net` | `docker network inspect bi-net --format '{{range .Containers}}{{.Name}} {{end}}'` |
| `password authentication failed for user "superset_ro"` | неверный пароль роли | пересоздать пароль через `ALTER ROLE` |
| `!override` не распознан | Compose старее 2.24.4 | `docker compose version` |
| страница не открывается | нет туннеля или Superset ещё стартует | `docker compose ps`, `curl -sI http://127.0.0.1:8088/health` на ВМ |

Не выполнять полный `docker inspect`, `docker compose config` без `--quiet` и `env` в контейнерах: они выводят пароли.

## 12. Состояние приёмки (по пунктам 8 ТЗ)

| Критерий | Статус |
|---|---|
| Версии и состав сервисов сверены с домашней машиной | по записям и по исходникам; команды сверки в `technical/home-stand.md`, раздел 10, ещё не выполнены |
| Все конфигурационные зависимости учтены | по исходникам: репозиторий 3.1.1 + одна правка + `compose.avalon.yaml` |
| Переменные описаны, секретов в комплекте нет | выполнено (в документе только заглушки) |
| Конфигурация проверена без раскрытия секретов | не проверено (`docker compose config --quiet` на ВМ) |
| Сети, mounts, права, restart-политики | по исходникам; `bi-net` новая, не проверена |
| Способ подключения Superset к учебной БД | предложен (`bi-net`), не проверен ни дома, ни на работе |
| Проверки подключения, графика и сохранности | подготовлены (раздел 8) |
| Первый и последующие запуски различаются | выполнено (разделы 7 и 9) |
| Перезагрузка и восстановление | не проверено |
| Итог передан файлами репозитория | выполнено после коммита этого файла пользователем |

## 13. Вероятные вопросы и ответы

Ответы подготовлены заранее. Если ответа нет или он не подходит к ситуации на ВМ, вопрос задаётся пользователю.

### Версии и происхождение

**Почему Superset 3.1.1, а не актуальная версия?**
Пользователь учится по курсу, снятому на 2.x–4.x. В 5.0 убраны устаревшие типы графиков, в 6.0 сменился интерфейс, и экран перестаёт совпадать с уроками. Домашний стенд тоже на 3.1.1. Переход на 6.x (встроенный MCP-сервис для этапа 2 с ИИ-агентами) — отдельная задача сразу для обоих стендов.

**`docker-compose-non-dev.yml` не предназначен для промышленной эксплуатации. Это допустимо?**
Да, для MVP на синтетике и показа руководству. **Риск:** перед переходом к постоянной работе с реальными данными нужна отдельная задача по усилению (HTTPS, резервные копии, мониторинг, обновления).

**Версии домашнего стенда не сверены с машиной. Это мешает развёртыванию?**
Нет. На ВМ образы скачиваются заново по тегам из раздела 1, домашние digest на результат не влияют. Сверка нужна для точности `technical/home-stand.md`; пользователь выполняет её отдельно.

**Фиксировать ли digest образов?**
После первого `docker compose pull` записать вывод `docker images --digests` в `technical/work-stand.md`. Подставлять digest в compose-файлы сейчас не нужно.

**Почему не скопировать репозиторий Superset в Avalon?**
Он большой, и его копия устареет. Воспроизводимость дают тег `3.1.1`, одна команда `sed` и файл `compose.avalon.yaml`. Если `git describe --tags` в каталоге superset показывает не `3.1.1`, остановиться и выяснить причину.

### Установка и сеть

**Ubuntu 26.04 нет в официальном репозитории Docker. Что делать?**
Не подставлять кодовое имя другой версии Ubuntu без согласования. Варианты для решения пользователем: дождаться поддержки; пакеты Ubuntu `docker.io` и `docker-compose-v2` (проверить, что Compose ≥ 2.24.4); репозиторий Docker для предыдущего LTS (риск несовместимости зависимостей). Рекомендация: пакеты Ubuntu, если версия Compose подходит.

**Образы или GitHub не скачиваются (корпоративный прокси, закрытый Docker Hub).**
Не обходить ограничения самостоятельно. Прокси для демона Docker настраивается по https://docs.docker.com/engine/daemon/proxy/; адрес прокси даёт пользователь, в репозиторий и `technical/` его не записывать (заглушка `<CORP_PROXY>`). Если Docker Hub закрыт, нужен внутренний реестр-зеркало — вопрос к администраторам через пользователя.

**Можно ли добавить пользователя в группу `docker`?**
Решает пользователь. Если ВМ личная и `sudo` уже есть, группа не добавляет новых прав. Иначе работать через `sudo`.

**Порт 5432 или 8088 на ВМ уже занят.**
Менять только левую часть публикации, например `127.0.0.1:15432:5432` или `127.0.0.1:18088:8088`, и тот же порт в SSH-туннеле. Внутренние порты и адрес `bi-postgres:5432` для Superset не меняются.

**SSH-туннель не нужен или невозможен.**
Если браузер запускается на самой ВМ, туннель не нужен: `http://localhost:8088`. Если SSH до ВМ закрыт, открывать порт в корпоративную сеть без отдельного решения пользователя нельзя.

**Что с автозапуском?**
Установка Docker из пакетов включает службу `docker` (проверка `systemctl is-enabled docker`). Контейнеры с `unless-stopped` поднимаются сами, если их не останавливали вручную. Перед запуском нужна существующая сеть `bi-net`.

### Конфигурация

**Работает ли переопределение `env_file` через `environment`?**
Да: по спецификации Compose `environment` имеет приоритет над `env_file`. Проверить без вывода значений: `docker compose config --quiet`, после запуска `docker exec superset_app sh -c 'test "$SUPERSET_SECRET_KEY" != TEST_NON_DEV_SECRET && echo OK'`.

**`!override` не поддерживается.**
Обновить Compose. Если нельзя, удалить `ports` из `compose.avalon.yaml` и вторым `sed` заменить в `docker-compose-non-dev.yml` строку `- 8088:8088` на `- "127.0.0.1:8088:8088"`; записать это как вторую правку исходников.

**Нужны ли `superset_worker` и `superset_worker_beat`?**
Для MVP не обязательны: они выполняют рассылки, алерты и асинхронные запросы. Оставлены, чтобы не отличаться от исходника. Рассылки реально не уходят: в конфигурации 3.1.1 `ALERT_REPORTS_NOTIFICATION_DRY_RUN = True`.

**Почему named volume, а не каталог на диске?**
Нет требований к отдельному разделу, не нужно настраивать владельцев и права. Физически тома лежат в `/var/lib/docker/volumes/`, места на ВМ достаточно.

**Кодировка и часовой пояс учебной базы?**
Образ `postgres:17` по умолчанию создаёт кластер в UTF8 с часовым поясом UTC. **Не проверено** на домашнем стенде. Для дат из 1С использовать `date` или `timestamptz`; на этом этапе ничего не настраивать.

**Пароль `admin` можно задать при инициализации?**
В 3.1.1 он зашит в `docker/docker-init.sh`. Сменить сразу после первого входа через интерфейс или командой `docker exec -it superset_app superset fab reset-password --username admin` (пароль вводится интерактивно и не попадает в журнал).

### Данные и безопасность

**Какие данные загружать на рабочий стенд?**
Для проверок — только синтетические (`contracts_demo` из раздела 8). Реальные выгрузки из 1С загружает пользователь сам; агент их не ищет, не читает и не выводит результаты запросов к ним. Агенту можно показывать только структуру таблиц и синтетические примеры (правило AGENTS.md).

**Куда положить файлы комплекта в репозитории?**
`infrastructure/postgres/compose.yaml`, `infrastructure/postgres/.env.example`, `infrastructure/superset/compose.avalon.yaml`, `infrastructure/superset/.env.example`. На ВМ они копируются в `<STAND_DIR>`. `.gitignore` Avalon уже исключает `.env` и `.env.*`, кроме `.env.example`, а также `*.dump`. Коммит делает пользователь.

**Что делать с домашним стендом?**
Ничего. Он не меняется в рамках этой задачи; перевод его на `bi-net` — отдельное решение пользователя.

**Как работать с ошибками развёртывания?**
Действовать короткими шагами, объяснять пользователю каждую команду, без раскрытия секретов в выводе. Исправления в комплекте фиксировать в этом файле или в `infrastructure/`, результаты проверок — в `docs/status.md`.

## 14. Итог для Codex

- **Подготовлено:** этот файл; исправленный `technical/home-stand.md`. Файлы из раздела 4 (compose, `.env.example`, `compose.avalon.yaml`) приведены целиком; разложить их в `infrastructure/` по указанным путям можно при развёртывании.
- **Фактически проверено:** только по записям установки дома и исходникам Superset 3.1.1. На машинах в этой задаче ничего не запускалось.
- **Выполняет пользователь:** генерация паролей и ключа, создание `.env`, ввод паролей в Superset, установка Docker через sudo.
- **Открытые вопросы:** версия Superset — сейчас 3.1.1 ради совпадения с учебным курсом; переход на 6.x (встроенный MCP для этапа 2) — отдельной задачей на обоих стендах. Политика компании насчёт группы `docker`.
- **Следующая задача:** развернуть стенд на рабочей ВМ по разделам 3–8 и записать результаты проверок в `docs/status.md`.
