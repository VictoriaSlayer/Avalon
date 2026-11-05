# Структура проекта Avalon

> Статус: актуален
> Обновлено: 2026-11-05 16:49 MSK
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
│   └── 01-stagename/
│       ├── task-01-name-task.md
│       ├── result-01-name-result.md
│       └── knowledge-01-name-knowledge.md
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
- project-stages/06-data-structure/task-01-data-description.md — шаблон
  метаописания данных и критерии завершения этапа 6.
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
