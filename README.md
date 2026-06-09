# Bitrix AI Knowledge Base

Единая база правил и навыков для AI-агентов Cursor при разработке на **1С-Битрикс**. Репозиторий клонируется в `.cursor/` Bitrix-проекта и задаёт агенту контекст: архитектуру `/local/`, стандарты D7, безопасность, кодстайл и порядок работы со скиллами.

**Репозиторий:** https://github.com/xesamanna2000/cursor

## Назначение

База знаний держит единый подход на разных проектах:

- архитектура `/local/` (модули, компоненты, роуты, миграции);
- PHP 8.2+, PSR-12, D7 API, безопасность;
- правила Cursor (`rules/*.mdc`) и тематические скиллы (`skills/*/SKILL.md`);
- планирование по скиллам, субагенты, code-review для нетривиальных диффов.

Обновление на всех проектах: `git pull` в `.cursor/`.

## Что внутри

Содержимое лежит **в корне** репозитория и после клонирования оказывается в `.cursor/`:

```text
.cursor/
├── AGENTS.md               # Главная инструкция для агента
├── README.md               # Это оглавление
├── rules/                  # 9 правил Cursor (.mdc)
│   ├── general-behavior.mdc
│   ├── plan-with-skills.mdc
│   ├── no-module-install.mdc
│   ├── no-bitrix-core.mdc
│   ├── no-superglobals.mdc
│   ├── always-include-module.mdc
│   ├── template-view-only.mdc      # globs: **/templates/**
│   ├── sprint-migration.mdc        # globs: local/migrations/**
│   └── no-debug-code.mdc           # globs: **/*.{php,js}
└── skills/                 # 32 тематических скилла
    ├── bitrix-modules/
    ├── bitrix-orm/
    ├── bitrix-sprint-migration/
    ├── bitrix-assets/
    └── ...
```

| Файл | Назначение |
| --- | --- |
| [`AGENTS.md`](AGENTS.md) | Роль агента, структура `/local/`, чек-лист, антипаттерны, таблица скиллов |
| `rules/*.mdc` | Жёсткие ограничения: `alwaysApply` или `globs` по типам файлов |
| `skills/*/SKILL.md` | Подробные инструкции по теме — читать перед реализацией |

## Подключение

### Первое подключение

Из корня Bitrix-проекта (рядом с `/local/`, `/bitrix/`):

```bash
git clone https://github.com/xesamanna2000/cursor.git .cursor
```

Cursor подхватит `.cursor/AGENTS.md`, `rules/` и `skills/`.

### Обновление

```bash
cd .cursor && git pull
```

### Submodule

```bash
git submodule add https://github.com/xesamanna2000/cursor.git .cursor
git submodule update --remote .cursor
```

## Как пользоваться

1. Клонировать в `.cursor/` целевого Bitrix-проекта.
2. Агент следует [`AGENTS.md`](AGENTS.md) и `rules/`.
3. По теме задачи — читать соответствующий `skills/<name>/SKILL.md` (таблица ниже).
4. Планирование — [`rules/plan-with-skills.mdc`](rules/plan-with-skills.mdc): скиллы → план → код → `code-reviewer` при нетривиальном диффе.
5. Правки инструкций — **в этом репозитории**, не в коде сайта.

## Правила Cursor

| Правило | Применение | Суть |
| --- | --- | --- |
| `general-behavior.mdc` | always | Русский язык, краткость, фокус на коде |
| `plan-with-skills.mdc` | always | План по скиллам, субагенты, ревью |
| `no-module-install.mdc` | always | Не править `/local/modules/*/install/**` |
| `no-bitrix-core.mdc` | always | Только `/local/`, не `/bitrix/` |
| `no-superglobals.mdc` | always | Не `$_GET`/`$_POST`/`$_SESSION` — Bitrix API |
| `always-include-module.mdc` | always | `Loader::includeModule()` перед классами модуля |
| `template-view-only.mdc` | globs: templates | `template.php` — рендер; логика в `class.php` |
| `no-debug-code.mdc` | globs: php, js | Без `var_dump`, `die`, `dd` в коммите |
| `sprint-migration.mdc` | globs: migrations | Схема через sprint.migration → скилл `bitrix-sprint-migration` |

Ключевые ограничения проекта:

- **`install/` модуля** — не редактировать; разработка в `lib/`, `admin/`, `.settings.php`.
- **Схема данных** — sprint.migration (если установлен); при задаче на HL/инфоблок сразу читать `bitrix-sprint-migration`.

## Планирование (`plan-with-skills.mdc`)

### Общий порядок

1. Скиллы по таблице в `AGENTS.md` → прочитать `SKILL.md`.
2. При незнакомом проекте — субагент `explore`.
3. План со ссылками на скиллы.
4. Реализация основным агентом.
5. `code-reviewer` — для нетривиальных диффов (несколько файлов, security, ORM, интеграции).

### Простая задача (< 8 ч, один файл)

Краткий план без оценки и `[model: …]`. `code-reviewer` — опционально.

### Сложная задача (> 8 ч, несколько подсистем)

В плане обязательно: оценка (часы/сложность), todos с `[model: slug]`, «промпт для старта», `code-reviewer` после кода.

### Субагенты

| Тип | Назначение |
| --- | --- |
| `explore` | Обзор `local/modules`, `routes`, компонентов |
| `shell` | `git`, `bitrix.php`, cron |
| `code-reviewer` | Сверка с `AGENTS.md`, скиллами, `rules/` |
| `ci-investigator` | Упавшие проверки в PR |

## Оглавление скиллов (32)

| Область | Скилл | Краткое описание |
| :--- | :--- | :--- |
| **Кодстайл** | `bitrix-psr-12` | PSR-12, strict_types, именование, шаблоны |
| **Модули** | `bitrix-modules` | Модуль без правок `install/`: `lib/`, `.settings.php`, `admin/` |
| **Миграции** | `bitrix-sprint-migration` | sprint.migration: HL, инфоблоки, UF, почта, DDL |
| **CLI** | `bitrix-console-commands` | `bitrix.php`, генераторы `make:*`, cron |
| **Контроллеры** | `bitrix-controllers` | Actions, ActionFilter, autowire, JSON |
| **Роутинг** | `bitrix-routing` | `web.php`, группы, `Router::route()`, urlrewrite |
| **ORM** | `bitrix-orm` | DataManager, query, fetchObject, события ORM |
| **События** | `bitrix-events` | `Event`, `EventResult`, `OnBefore*` |
| **Валидация** | `bitrix-validation` | ValidationService, атрибуты, DTO |
| **DI** | `bitrix-service-locator` | ServiceLocator, регистрация сервисов |
| **Кеширование** | `bitrix-caching` | Cache, TaggedCache, ManagedCache |
| **Безопасность** | `bitrix-security` | CSRF, ORM filter, XSS, права |
| **Фон/Задачи** | `bitrix-background-jobs` | CAgent, addBackgroundJob, Messenger |
| **Результаты** | `bitrix-result-and-errors` | Result, Error, ErrorCollection |
| **Компоненты** | `bitrix-components` | class.php, кеш, Controllerable, SEF |
| **Инфоблоки** | `bitrix-iblocks` | IblockTable, свойства, CIBlock* |
| **HL-блоки** | `bitrix-highloadblocks` | HighloadBlockTable, compileEntity, права |
| **Пользователи** | `bitrix-user` | UserTable, CUser, $USER, группы, UF_* |
| **Прямой SQL** | `bitrix-database` | Connection, SqlExpression, транзакции |
| **HTTP-слой** | `bitrix-request-response` | Context, HttpRequest, Json, Redirect |
| **HTTP-клиент** | `bitrix-http-client` | HttpClient, async, SSRF |
| **Сессии** | `bitrix-sessions` | getSession, kernel/local session |
| **Дата/Время** | `bitrix-datetime` | Date, DateTime, таймзоны |
| **Локализация** | `bitrix-localization` | Loc, lang-файлы, BX.message |
| **Логирование** | `bitrix-logger` | PSR-3, FileLogger, loggers |
| **Ассеты** | `bitrix-assets` | Asset, Extension::load, addJs/addCss |
| **JS** | `bitrix-js` | BX, BX.ajax.runAction, BX.namespace |
| **Админка** | `bitrix-admin` | CAdminList, CAdminTabControl, admin/ |
| **Composite** | `bitrix-composite` | createFrame, FrameStatic, динамические области |
| **Почта** | `bitrix-mail` | Mail\Event::send(), шаблоны, миграции |
| **Файлы** | `bitrix-file` | CFile, FileField, IO |
| **SEO** | `bitrix-seo` | SetTitle, мета, canonical, sitemap |

Полная таблица с привязкой к задачам — в [`AGENTS.md`](AGENTS.md#когда-какой-скилл-использовать).

## Что сюда не входит

- Код конкретного сайта (`/local/` проекта).
- Секреты, `.env`, зависимости Composer/npm проекта.

## Требования к окружению

- Bitrix **23.0+** (целевой — **25.x**);
- PHP **8.2+**;
- D7, Composer, `bitrix.php`, при необходимости sprint.migration.

Подробности — в [`AGENTS.md`](AGENTS.md).
