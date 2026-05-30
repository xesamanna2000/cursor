# Bitrix AI Knowledge Base

Единая база правил и навыков для AI-агентов Cursor при разработке на **1С-Битрикс**. Репозиторий подключается в каталог `.cursor/` Bitrix-проекта и задаёт агенту общий контекст: архитектуру `/local/`, стандарты D7, безопасность, кодстайл и порядок работы со скиллами.

**Репозиторий:** https://github.com/xesamanna2000/cursor

## Назначение

База знаний помогает держать единый подход на разных проектах:

- архитектуру и структуру `/local/`;
- стандарты PHP 8.2+, PSR-12, D7 API и безопасной разработки;
- правила общения и планирования в Cursor;
- типовые сценарии: модули, ORM, контроллеры, роутинг, кеш, события, фоновые задачи и т.д.

Вместо ручного копирования инструкций используется один репозиторий на GitHub. Его можно клонировать или подключить как submodule, а затем централизованно обновлять через `git pull`.

## Что внутри

Содержимое репозитория лежит **в корне** и после клонирования оказывается в `.cursor/` целевого Bitrix-проекта:

```text
.cursor/                    ← каталог после подключения к Bitrix-проекту
├── AGENTS.md               # Главная инструкция для агента
├── README.md               # Описание базы знаний и оглавление скиллов
├── rules/                  # Правила Cursor (.mdc)
│   ├── general-behavior.mdc
│   ├── no-module-install.mdc
│   └── plan-with-skills.mdc
└── skills/                 # Тематические навыки Cursor
    ├── bitrix-psr-12/
    ├── bitrix-orm/
    ├── bitrix-modules/
    └── ...
```

| Файл | Назначение |
| --- | --- |
| `AGENTS.md` | Роль агента, структура проекта, антипаттерны, таблица «когда какой скилл использовать» |
| `rules/*.mdc` | Правила Cursor с `alwaysApply` или привязкой к файлам — дополняют `AGENTS.md` |
| `skills/*/SKILL.md` | Подробные инструкции по конкретной теме — подгружаются агентом по задаче |

## Подключение

### Первое подключение

Из корня Bitrix-проекта (рядом с `/local/`, `/bitrix/`):

```bash
git clone https://github.com/xesamanna2000/cursor.git .cursor
```

После клонирования Cursor автоматически подхватит `.cursor/AGENTS.md`, правила из `.cursor/rules/` и скиллы из `.cursor/skills/`.

### Обновление

```bash
cd .cursor
git pull
```

### Вариант через submodule (если проект уже в Git)

Удобно, когда нужно зафиксировать версию базы знаний в истории проекта:

```bash
git submodule add https://github.com/xesamanna2000/cursor.git .cursor
git commit -m "Подключить Bitrix AI Knowledge Base"
```

Обновление:

```bash
git submodule update --remote .cursor
```

## Как пользоваться в работе

1. Подключить репозиторий в `.cursor/` целевого Bitrix-проекта (см. выше).
2. Открыть Bitrix-проект в Cursor — агент будет следовать `.cursor/AGENTS.md` и правилам из `.cursor/rules/`.
3. При задачах по конкретной теме агент обращается к соответствующему скиллу из `.cursor/skills/`.
4. При планировании и реализации — правило [`rules/plan-with-skills.mdc`](rules/plan-with-skills.mdc): чтение релевантных `SKILL.md`, при необходимости разведка субагентами, для сложных задач — оценка и маршрутизация моделей, после кода — `code-reviewer` (см. раздел ниже).
5. Изменения в правилах и скиллах вносятся **здесь**, в этом репозитории, а не в коде сайта.
6. На каждом проекте периодически выполнять `git pull` в `.cursor/`, чтобы получить актуальные инструкции.

## Правила Cursor

| Правило | Назначение |
| --- | --- |
| `general-behavior.mdc` | Общий стиль общения: русский язык, краткость, фокус на коде |
| `plan-with-skills.mdc` | Планирование и исполнение по скиллам — см. [Планирование](#планирование-plan-with-skillsmdc) |
| `no-module-install.mdc` | Запрет на создание и изменение `install/` внутри Bitrix-модулей проекта |

## Планирование (`plan-with-skills.mdc`)

Правило с `alwaysApply: true` задаёт порядок работы агента при Plan mode, todo-листах и пошаговой реализации.

### Общий порядок

1. Выбрать скиллы по таблице в `AGENTS.md` → прочитать `skills/<name>/SKILL.md`.
2. При незнакомом проекте — субагент `explore` (разведка без правок кода).
3. Собрать план со ссылками на скиллы и маркерами шагов.
4. Реализовать основным агентом (код и финальные правки — не отдавать целиком субагенту).
5. После нетривиального диффа — обязательный `code-reviewer` до ответа пользователю.

### Простая задача

Один компонент/файл, оценка **< 8 ч** — краткий план: скиллы, шаги, антипаттерны. Оценка в часах, `[model: …]` и «промпт для старта» не обязательны; ревью — по ситуации.

### Сложная задача

Несколько подсистем, нетривиальная архитектура или **> 8 ч** — в план **обязательно** три блока (дополняют субагентов, не заменяют):

| Блок | Содержание |
| --- | --- |
| Оценка | Таблица **часы / сложность** (`низкая` … `очень высокая`), не «дни» |
| Маршрутизация | Todos с `[model: slug]` (Cursor Task) и при необходимости `[субагент: explore]` / `[субагент: code-reviewer]` |
| Промпт для старта | Готовый текст для нового Agent-чата: путь к плану, порядок todos, skills, `/local/`, стартовый шаг, обязательное ревью после кода |

Модели для todos (slug из Cursor Task): `claude-opus-4-8-thinking-high` (ядро, интеграции), `composer-2.5-fast` (шаблоны, wiring, docs), `gemini-3.1-pro` / `gpt-5.5-medium` — альтернативы по объёму и сложности.

### Субагенты

| Тип | Назначение |
| --- | --- |
| `explore` | Обзор `local/modules`, `routes`, компонентов перед планом |
| `generalPurpose` | Многошаговое исследование вне простого поиска |
| `shell` | `git`, `bitrix.php`, cron |
| `code-reviewer` | Сверка диффа с `AGENTS.md`, скиллами из плана и `rules/*.mdc` |
| `ci-investigator` | Упавшие проверки в PR |

Полный текст правила: [`rules/plan-with-skills.mdc`](rules/plan-with-skills.mdc).

## Оглавление скиллов

| Область | Скилл | Краткое описание |
| :--- | :--- | :--- |
| **Кодстайл** | `bitrix-psr-12` | PSR-12, strict_types, именование, типизация, шаблоны |
| **Модули** | `bitrix-modules` | Создание модуля, `install/index.php`, регистрация в админке |
| **CLI** | `bitrix-console-commands` | CLI, `bitrix.php`, генераторы `make:*`, cron, своя команда |
| **Контроллеры** | `bitrix-controllers` | AJAX/REST-контроллеры, actions, фильтры, autowire, render |
| **Роутинг** | `bitrix-routing` | Маршруты, группы, префиксы, генерация URL, миграция с urlrewrite |
| **ORM** | `bitrix-orm` | `DataManager`, `getMap`, `query()`, `fetchObject`, события ORM |
| **События** | `bitrix-events` | Новые события (`Event`, `EventResult`) и старые `OnBefore*` |
| **Валидация** | `bitrix-validation` | Валидация объектов/DTO, атрибуты, `ValidationService` |
| **DI** | `bitrix-service-locator` | `ServiceLocator`, DI, регистрация сервисов |
| **Кеширование** | `bitrix-caching` | `Cache`, `ManagedCache`, `TaggedCache`, сброс по таблице |
| **Безопасность** | `bitrix-security` | CSRF, SQLi-риски ORM, XSS, права |
| **Фон/Задачи** | `bitrix-background-jobs` | Агенты, `addBackgroundJob`, очереди `Messenger` |
| **Результаты** | `bitrix-result-and-errors` | `Result`, `Error`, `ErrorCollection`, типизированные результаты |
| **Компоненты** | `bitrix-components` | Компоненты: `class.php`, шаблоны, кеш, `Controllerable`, AJAX |
| **Инфоблоки** | `bitrix-iblocks` | Инфоблоки: `IblockTable`, свойства, SEO, `CIBlock*`-API |
| **HTTP-клиент** | `bitrix-http-client` | `HttpClient`: legacy/PSR-18, async, прокси, SSRF |
| **Логирование** | `bitrix-logger` | PSR-3 логи: `FileLogger`, `LogFormatter`, секция `loggers` |
| **Локализация** | `bitrix-localization` | `Loc::getMessage`, lang-файлы, `Culture`, `BX.message` |
| **Дата/Время** | `bitrix-datetime` | `Bitrix\Main\Type\Date/DateTime`, таймзоны, `toUserTime` |
| **HTTP-слой** | `bitrix-request-response` | `Application`, `Context`, `HttpRequest`, `HttpResponse`, `Json`/`Redirect` |
| **Сессии** | `bitrix-sessions` | Сессии: `getSession`, `getKernelSession`, `getLocalSession`, режимы |
| **Прямой SQL** | `bitrix-database` | `Connection`, `SqlHelper`, `SqlExpression`, транзакции, DDL |
| **Спец. задание** | `bitrix-ai-challenge` | Требования и ограничения модуля «Избранное» (ТЗ) |

## Что сюда не входит

- Код конкретного сайта (`/local/`, модули, компоненты).
- Настройки окружения, секреты, `.env`.
- Зависимости Composer/npm проекта.

Этот репозиторий — только **инструкции для AI**, общие для всех Bitrix-проектов команды.

## Требования к окружению

Скиллы ориентированы на:

- Bitrix **23.0+** (целевой — **25.x**);
- PHP **8.2+**;
- современное ядро D7, Composer, консоль `bitrix.php`.

Подробности — в [AGENTS.md](AGENTS.md).
