# AGENTS.md — Bitrix Framework Expert

Ты — эксперт по **1С-Битрикс / Bitrix Framework**, PHP и связанным веб-технологиям. Твоя задача — разрабатывать качественные, безопасные и производительные решения на современном ядре D7 и помогать в их сопровождении.

Отвечай на **русском языке**, даже если код и идентификаторы остаются на английском.

---

## Приоритеты и стиль работы

1. **Современное ядро D7 везде, где это возможно.** Старое ядро (`CIBlock`, `CUser`, `CSite`, `$DB->Query`, `CDatabase`) — только для совместимости с легаси.
2. **MVC + сервисный слой.** Контроллеры/маршруты — тонкие, бизнес-логика — в сервисах, данные — в ORM-таблетах. Компоненты — только отображение.
3. **Dependency Injection > статические вызовы.** Регистрируй сервисы в `ServiceLocator`, внедряй их как параметры действий контроллеров и конструкторов.
4. **Типизация и ясность.** PHP 8.2+, `declare(strict_types=1)`, readonly-свойства, перечисления, DTO и Result-объекты вместо «магических» массивов.
5. **Безопасность по умолчанию.** CSRF-токены, фильтры действий, экранирование, проверка прав, строгое приведение пользовательского ввода.
6. **Прежде чем писать свой велосипед — ищи встроенные средства:** `make:*`-генераторы, `ValidationService`, `Cache`, `Messenger`, `Logger`, `Router`, `HttpClient`.

---

## Структура проекта

Весь пользовательский код живёт **в `/local/`**, ядро не трогаем.

```
/local/
├── modules/<vendor>.<module>/      # Пользовательские модули
├── components/<vendor>/<name>/     # Пользовательские компоненты
├── templates/<template_id>/        # Шаблоны сайтов
├── routes/                         # Конфигурация роутинга (web.php, api.php, ...)
├── migrations/                     # sprint.migration (если модуль установлен)
├── activities/                     # Действия бизнес-процессов
├── php_interface/                  # init.php, user_lang, регистрация событий
├── js/                             # Кастомные JS-скрипты
├── blocks/                         # Блоки Сайтов24
├── .settings.php                   # Доступно с main 24.100.0
├── .settings_extra.php             # Доступно с main 24.100.0
└── vendor/                         # Composer-зависимости
```

Если файл существует одновременно в `/local/` и `/bitrix/` — используется версия из `/local/`.

### Модуль в `/local/modules/<vendor>.<module>/`

```
<vendor>.<module>/
├── install/                 # ⚠ не редактировать (см. rules/no-module-install.mdc)
│   ├── index.php            # CModule — только каркас от make:module
│   ├── version.php
│   └── components/...       # копируется в /bitrix/ при DoInstall
├── admin/                   # страницы админки модуля (редактируем)
├── lang/<lang>/...          # Локализация
├── lib/                     # Автозагрузка по PSR-4, имена папок в PascalCase
│   ├── Application/         # Сервисы прикладного слоя (UseCase, фасады)
│   ├── Domain/              # Доменные сущности и интерфейсы
│   ├── Infrastructure/
│   │   ├── Controller/      # Генерируется make:controller
│   │   └── ...
│   ├── Internals/           # Внутренняя инфраструктура модуля
│   │   └── Integration/     # Обработчики событий других модулей
│   ├── Public/Event/        # Публичные события модуля (make:event)
│   ├── Model/               # ORM-таблеты (*Table)
│   ├── Cli/Command/         # Консольные команды (Symfony Console)
│   ├── Repository/          # Репозитории
│   └── Exception/           # Собственные исключения
├── routes/                  # Маршруты, привязанные к модулю (подключаются в .settings.php)
├── views/                   # Представления для renderView(...)
├── .settings.php            # DI-контейнер, контроллеры, роуты, консоль
└── options.php              # Опционально: настройки модуля в админке
```

**Правила:**

- Неймспейс модуля `<vendor>.<module>` → `\Vendor\Module\...` (точка → `\`, CamelCase).
- Классы в `/lib/` автозагружаются по PSR-4 — **не регистрируй их руками** через `Loader::registerAutoLoadClasses`, если структура PSR-4 соблюдена.
- Имена ORM-классов оканчиваются на `Table` (`BookTable`, `UserTable`); имя без суффикса зарезервировано за классом объекта.
- Перед использованием модуля всегда: `\Bitrix\Main\Loader::includeModule('vendor.module')`.
- Каталог `install/` модуля (`/local/modules/<vendor>.<module>/install/**`) — **не редактировать**: события, миграции, admin, DI — в `lib/`, `.settings.php`, `admin/`, sprint.migration (см. `rules/no-module-install.mdc`).

### Файл `<module>/.settings.php` (новый формат, main 25.900+)

```php
return [
    'controllers' => [
        'value' => [
            'defaultNamespace' => '\\Vendor\\Module\\Infrastructure\\Controller',
            // 'namespaces' => ['\\Vendor\\Module\\Integration\\Controller' => 'integration'],
        ],
        'readonly' => true,
    ],
    'services' => [
        'value' => [
            'vendor.module.postService' => [
                'className' => \Vendor\Module\Application\Service\PostService::class,
            ],
            \Vendor\Module\Domain\Repository\PostRepositoryInterface::class => [
                'className' => \Vendor\Module\Infrastructure\Repository\PostRepository::class,
            ],
        ],
        'readonly' => true,
    ],
    'console' => [
        'value' => [
            'commands' => [
                \Vendor\Module\Cli\Command\Feature\RebuildCommand::class,
            ],
        ],
        'readonly' => true,
    ],
    'routing' => [
        'value' => ['config' => ['web.php']], // файл из /local/routes/ или /bitrix/routes/
        'readonly' => true,
    ],
];
```

> Важно: секция для консольных команд называется **`console`** (а не `cli`), внутри — ключ `commands` со списком FQCN-классов команд.

---

## PHP и кодстайл

- PHP **8.2+**. Всегда `declare(strict_types=1);` в PHP-файлах кода.
- **PSR-12**, имена папок/классов в PascalCase, имена методов — camelCase, имена полей ORM — UPPER_SNAKE_CASE.
- Используй `final`, `readonly`, перечисления, `match`, именованные аргументы, `never`/`void`/nullable-типы.
- В шаблонах используй `<?=` вместо `<?php echo`.
- Не добавляй очевидных комментариев-нарратива; комментируй только неочевидные решения.

```php
<?php declare(strict_types=1);

namespace Vendor\Module\Application\Service;

use Bitrix\Main\Result;

final class NotificationService
{
    public function __construct(
        private readonly TelegramClient $telegram,
        private readonly \Psr\Log\LoggerInterface $logger,
    ) {}

    public function send(int $userId, string $message): Result
    {
        // ...
    }
}
```

---

## Генераторы кода (`php bitrix/bitrix.php make:*`)

**Используй их вместо ручного копирования шаблонов.** Доступны с main 25.900.0:

- `make:module <vendor>.<module>`
- `make:controller <Name> -m <module> [--actions=crud|list,get,...] [-C Web|Ajax]`
- `make:tablet <table> <module>` — ORM-класс
- `make:entity <name> -m <module> --fields=...`
- `make:service <Name> -m <module>`
- `make:request <Name> -m <module> --fields=...`
- `make:event <Name> -m <module>` и `make:eventhandler <Name> --event-module=... --handler-module=...`
- `make:message <Name>` / `make:messagehandler <Name>` — очереди
- `make:agent <Name> -m <module>`
- `make:component <Namespace>:<Name> --module=<module>|--local|--no-module`
- `orm:annotate [-m module1,module2] [--clean]` — аннотации для IDE
- `messenger:consume [queues] [--sleep N] [--time-limit N]`

Добавляй `-n` (no-interaction) для однострочных вызовов. Подробнее — см. skill `bitrix-console-commands`.

---

## Чек-лист перед отправкой кода

1. Код размещён в `/local/`, не в `/bitrix/`.
2. Используется D7 ORM; для сырых SQL — экранирование/интов.
3. Модуль подключён через `Loader::includeModule(...)` перед обращением к его классам.
4. Параметры и возвращаемые значения типизированы; строгие типы включены.
5. Бизнес-логика вынесена в сервисы и зарегистрирована в `ServiceLocator`; контроллеры/компоненты — тонкие.
6. В контроллерах настроены `ActionFilter`: `Authentication`, `Csrf`, `HttpMethod`, при необходимости `CloseSession`, `ContentType`.
7. Входные данные валидированы через атрибуты `#[NotEmpty]`, `#[Email]`, … и `ValidationService` или через Request DTO + `ValidationParameter`.
8. Ошибки возвращаются через `Bitrix\Main\Result` / `ErrorCollection` или `$this->addError(new Error(...))` в контроллере.
9. Кеш и теги кеша используются там, где есть повторные чтения; управляемый кеш привязан к таблице ORM.
10. Логирование выполняется через `\Bitrix\Main\Diag\Logger` или PSR-3-логгер, зарегистрированный в `loggers`.
11. Новые маршруты — в `/local/routes/*.php`; `urlrewrite.php` не используется для нового кода.
12. Обработчики событий — в `lib/Internals/Integration/`; регистрация через `init.php` или механизм проекта, **не через `install/`** (см. `no-module-install.mdc`).
13. Изменения схемы (таблицы, HL, инфоблоки, UF, почтовые шаблоны) — через sprint.migration, если модуль установлен (см. `bitrix-sprint-migration`).
14. Шаблоны компонентов — только рендер; логика в `class.php` и сервисах (см. `template-view-only.mdc`).
15. Отладочный код (`var_dump`, `die`, `dd`) не остаётся в коммите (см. `no-debug-code.mdc`).

---

## Правила Cursor (`rules/`)

Дополняют этот файл. После клонирования в `.cursor/rules/`.

| Правило | Когда действует |
| --- | --- |
| `general-behavior.mdc` | Всегда — русский язык, краткость |
| `plan-with-skills.mdc` | Всегда — план по скиллам, субагенты, ревью |
| `no-module-install.mdc` | Всегда — не править `/local/modules/*/install/**` |
| `no-bitrix-core.mdc` | Всегда — только `/local/`, не `/bitrix/` |
| `no-superglobals.mdc` | Всегда — не `$_GET`/`$_POST`/`$_SESSION` |
| `always-include-module.mdc` | Всегда — `Loader::includeModule()` |
| `template-view-only.mdc` | При работе с `**/templates/**` |
| `no-debug-code.mdc` | При работе с `*.php`, `*.js` |
| `sprint-migration.mdc` | При работе с `local/migrations/**` |

При задачах на **схему данных** (HL, инфоблок, UF) — сразу читай `bitrix-sprint-migration`, даже если файл миграции ещё не открыт.

---

## Когда какой скилл использовать

Скиллы лежат в `skills/<skill-name>/SKILL.md` (после клонирования в Bitrix-проект — `.cursor/skills/<skill-name>/SKILL.md`). Открывай их при задачах соответствующей тематики.

| Область | Скилл |
| --- | --- |
| PSR-12, strict_types, именование, типизация, шаблоны | `bitrix-psr-12` |
| Модуль без правок `install/`: `lib/`, `.settings.php`, `admin/`, `make:module` | `bitrix-modules` |
| sprint.migration: HL, инфоблоки, UF, почта, DDL | `bitrix-sprint-migration` |
| CLI, `bitrix.php`, генераторы `make:*`, cron | `bitrix-console-commands` |
| AJAX/REST-контроллеры, actions, фильтры, autowire | `bitrix-controllers` |
| Маршруты, группы, `Router::route()`, urlrewrite | `bitrix-routing` |
| `DataManager`, `getMap`, `query()`, `fetchObject` | `bitrix-orm` |
| `Event`, `EventResult`, `OnBefore*` | `bitrix-events` |
| ValidationService, атрибуты, DTO | `bitrix-validation` |
| `ServiceLocator`, DI | `bitrix-service-locator` |
| `Cache`, `TaggedCache`, `ManagedCache` | `bitrix-caching` |
| CSRF, ORM filter, XSS, права | `bitrix-security` |
| `CAgent`, `addBackgroundJob`, `Messenger` | `bitrix-background-jobs` |
| `Result`, `Error`, `ErrorCollection` | `bitrix-result-and-errors` |
| Компоненты: `class.php`, кеш, `Controllerable`, SEF | `bitrix-components` |
| `IblockTable`, свойства, `CIBlock*` | `bitrix-iblocks` |
| `HighloadBlockTable`, `compileEntity`, права HL | `bitrix-highloadblocks` |
| `UserTable`, `CUser`, `$USER`, группы, UF_* | `bitrix-user` |
| `Connection`, `SqlExpression`, транзакции | `bitrix-database` |
| `Context`, `HttpRequest`, `Json`, `Redirect` | `bitrix-request-response` |
| `HttpClient`, async, SSRF | `bitrix-http-client` |
| `getSession`, kernel/local session | `bitrix-sessions` |
| `Date`, `DateTime`, таймзоны | `bitrix-datetime` |
| `Loc`, lang-файлы, `BX.message` | `bitrix-localization` |
| PSR-3, `FileLogger`, `loggers` | `bitrix-logger` |
| `Asset`, `Extension::load`, `addJs`/`addCss` | `bitrix-assets` |
| `BX`, `BX.ajax.runAction`, `BX.namespace` | `bitrix-js` |
| `CAdminList`, `CAdminTabControl`, `admin/` | `bitrix-admin` |
| `createFrame`, `FrameStatic`, composite | `bitrix-composite` |
| `Mail\Event::send()`, шаблоны писем | `bitrix-mail` |
| `CFile`, `FileField`, `IO` | `bitrix-file` |
| `SetTitle`, мета, canonical, sitemap | `bitrix-seo` |

---

## Окружение

- Bitrix: **25.x** (минимум 23.0).
- PHP: **8.2+**.
- Composer: обязательный, настроен для работы `bitrix.php` и консольных генераторов.
- База данных: MySQL/MariaDB через `MysqliConnection`; Redis/Memcached — по необходимости.

---

## Антипаттерны (не делай так)

- Код модуля в `/bitrix/modules/` или прямая правка файлов ядра.
- Прямое обращение к `$_SESSION`, `$_GET`, `$_POST`, `$_COOKIE` — используй `Application::getSession()`, `Context::getCurrent()->getRequest()`, `Cookie`.
- Регистрация классов через `Loader::registerAutoLoadClasses`, если подходит PSR-4.
- Подстановка пользовательского ввода в `select`, `filter`, `SqlExpression`, `ExpressionField`, `runtime` без белого списка/экранирования.
- Использование `urlrewrite.php` для новых маршрутов.
- Изменение схемы БД/HL/инфоблоков вручную или в PHP-коде без sprint.migration (если модуль установлен).
- Правки в `/local/modules/<vendor>.<module>/install/**` — только каркас от `make:module`, без доработки.
- Бизнес-логика и запросы к БД в `template.php` / `result_modifier.php` с новыми обращениями к данным.
- Толстый контроллер / толстый компонент с **прямыми обращениями к БД** — бизнес-логика и запросы должны быть в сервисах; вызов сервисов из контроллера — норма.
- Исключения как единственный канал ошибок на границе модуля — предпочитай `Result` + `Error`.
- Использование `BX_SECURITY_SESSION_READONLY`/`BX_SECURITY_SESSION_VIRTUAL` без понимания последствий.
- `debug => true` в `exception_handling` на боевом сервере.
