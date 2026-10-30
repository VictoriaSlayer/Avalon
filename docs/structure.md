# Структура проекта Avalon

> Статус: актуален
> Обновлено: 2026-10-30 18:56 MSK
> Ответственный: Codex

## Фактическая структура

```text
Avalon/
├── README.md
├── AGENTS.md
├── CLAUDE.md
├── .gitignore
├── Необходимо описать окружение и версии.md
├── docs/
│   ├── structure.md
│   ├── plan.md
│   ├── status.md
│   └── decisions.md
├── project-stages/
│   └── 01-make-stand/
│       ├── task-01-make-stand.md
│       ├── result-01-stand-config.md
│       ├── task-02-make-stand.md
│       ├── incident-01-superset-access.md
│       └── knowledge-01-linux-comands.md
├── technical/
│   ├── home-stand.md
│   ├── home-stand-commands.md
│   └── work-stand.md
└── infrastructure/
    ├── README.md
    ├── postgres/
    │   ├── compose.yaml
    │   └── .env.example
    └── superset/
        ├── compose.avalon.yaml
        └── .env.example
```

## Назначение

- README.md — цели и контекст проекта.
- AGENTS.md — единые правила; CLAUDE.md — точка входа для Claude.
- docs/plan.md — этапы; status.md — актуальное состояние;
  decisions.md — существенные решения и открытые вопросы.
- project-stages/ — задачи, результаты, диагностика и знания по этапам.
  Отдельных папок tasks/, results/ и knowledge/ сейчас нет.
  Задачи описываются до выполнения и обновляются по ходу.
- technical/ — описания стендов и обслуживание; чувствительные пути,
  учётные сведения и сведения о дисках заменяются заглушками.
- infrastructure/ — актуальные эталонные конфигурации без секретов.
  Примеры внутри исходного результата Claude не заменяют эти файлы.

## Рабочие файлы вне репозитория

Фактический запуск выполняется из ~/bi-stand/ с локальными .env.
Они и содержимое хранилищ Docker в Git не включаются.
Общие изменения между эталоном и рабочими копиями переносятся явно.
Папки datasets/ и backups/ в репозитории не предусмотрены.

## Будущее расширение

sql/schemas/, sql/transformations/ и sql/checks/ создаются
по мере появления рабочих SQL-файлов. Пока этих каталогов нет;
синтетическая таблица для проверки прав создана пользователем в БД.
Не создавать пустую структуру заранее без задачи.
