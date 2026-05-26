# Bitrix AI Knowledge Base

Единый набор инструкций и навыков для AI-агентов (Cursor) при разработке на **1С-Битрикс**. Репозиторий не является частью конкретного сайта — это переиспользуемая база знаний, которую подключают к любому Bitrix-проекту.

**Репозиторий:** https://github.com/xesamanna2000/cursor

## Зачем это нужно

На каждом Bitrix-проекте агент должен одинаково понимать:

- архитектуру и структуру `/local/`;
- стандарты кода (D7, PSR-12, безопасность);
- типовые задачи (модули, ORM, контроллеры, кеш, события и т.д.).

Вместо копирования правил и скиллов вручную — один репозиторий на GitHub, который клонируется в каталог `.cursor/` каждого проекта. Обновления правил делаются централизованно и подтягиваются через `git pull`.

## Что внутри

Содержимое репозитория лежит **в корне** и после клонирования оказывается в `.cursor/` целевого Bitrix-проекта:

```
.cursor/               ← каталог после git clone в Bitrix-проекте
├── AGENTS.md          # Главная инструкция для агента: приоритеты, структура, чек-листы
├── README.md          # Этот файл — описание базы знаний и оглавление скиллов
└── skills/            # Тематические навыки (SKILL.md по каждой области Bitrix)
    ├── bitrix-psr-12/
    ├── bitrix-orm/
    ├── bitrix-modules/
    └── ...
```

| Файл | Назначение |
| --- | --- |
| `AGENTS.md` | Роль агента, структура проекта, антипаттерны, таблица «когда какой скилл использовать» |
| `skills/*/SKILL.md` | Подробные инструкции по конкретной теме — подгружаются агентом по задаче |

## Как подключить к Bitrix-проекту

### Первое подключение

Из корня Bitrix-проекта (рядом с `/local/`, `/bitrix/`):

```bash
git clone https://github.com/xesamanna2000/cursor.git .cursor
```

После клонирования Cursor автоматически подхватит `.cursor/AGENTS.md` и скиллы из `.cursor/skills/`.

### Обновление правил

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
2. Открыть Bitrix-проект в Cursor — агент будет следовать `.cursor/AGENTS.md`.
3. При задачах по конкретной теме агент обращается к соответствующему скиллу из `.cursor/skills/`.
4. Изменения в правилах и скиллах вносятся **здесь**, в этом репозитории, а не в коде сайта.
5. На каждом проекте периодически выполнять `git pull` в `.cursor/`, чтобы получить актуальные инструкции.

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
