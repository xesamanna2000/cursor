---
name: bitrix-sprint-migration
description: Покрывает sprint.migration (andreyryabin/sprint.migration) — создание и применение миграций, helpers для HL-блоков, инфоблоков, UF-полей, почтовых событий, SQL. Применяется при изменении схемы БД, справочников и структуры данных на проектах с установленным sprint.migration. Ключевые термины — sprint.migration, Version, getHelperManager, saveHlblock, saveIblock, migrate.php.
---

# sprint.migration

Модуль `andreyryabin/sprint.migration`. На проектах с ним **все изменения структуры данных** — только через миграции в `/local/migrations/`, не вручную в админке и не в `install/` модуля.

Если модуль **не установлен** — уточни у пользователя альтернативу; не предлагай правки в `install/`.

## Что оформлять миграцией

- DDL: таблицы, индексы, колонки
- HL-блоки и поля `UF_*`
- Инфоблоки, типы, свойства
- Пользовательские поля (`USER_TYPE_ENTITY`)
- Почтовые события и шаблоны
- Права на инфоблоки, HL-блоки, разделы

## Создание миграции

```bash
php /bitrix/modules/sprint.migration/tools/migrate.php add "Описание изменения"
```

Файл появится в `/local/migrations/` (или путь из настроек модуля).

## Структура файла

```php
namespace Sprint\Migration;

class Version20260609120000 extends Version
{
    protected $description = 'Создать HL-блок Color и его поля';

    public function up(): void
    {
        $helper = $this->getHelperManager();

        $helper->Hlblock()->saveHlblock([
            'NAME'       => 'Color',
            'TABLE_NAME' => 'b_hlblock_color',
        ]);

        $helper->Hlblock()->saveField('Color', [
            'FIELD_NAME'        => 'UF_NAME',
            'USER_TYPE_ID'      => 'string',
            'MANDATORY'         => 'Y',
            'SORT'              => 100,
            'EDIT_FORM_LABEL'   => ['ru' => 'Название'],
        ]);

        $this->outSuccess('HL-блок Color создан');
    }

    public function down(): void
    {
        $helper = $this->getHelperManager();
        $helper->Hlblock()->deleteHlblock('Color');
        $this->outSuccess('HL-блок Color удалён');
    }
}
```

## Ключевые helpers

| Helper | Для чего |
|---|---|
| `$helper->Hlblock()` | `saveHlblock()`, `saveField()`, `deleteHlblock()`, `saveGroupPermissions()` |
| `$helper->Iblock()` | `saveIblock()`, `saveProperty()`, `saveIblockType()` |
| `$helper->UserTypeEntity()` | `saveUserTypeEntity()`, `deleteUserTypeEntitiesIfExists()` |
| `$helper->Event()` | `saveEventType()`, `saveEventMessage()` |
| `$helper->Sql()` | `$helper->Sql()->query('ALTER TABLE ...')` |
| `$helper->UserGroup()` | `$helper->UserGroup()->getGroupId('CODE')` |

## Применение

```bash
php /bitrix/modules/sprint.migration/tools/migrate.php up
php /bitrix/modules/sprint.migration/tools/migrate.php up Version20260609120000
php /bitrix/modules/sprint.migration/tools/migrate.php down Version20260609120000
php /bitrix/modules/sprint.migration/tools/migrate.php ls
```

## Правила написания

- **Всегда реализуй `down()`** — откат обязателен.
- **`save*` вместо `add`** — идемпотентность: создаст или обновит.
- **Не хардкодить ID** — `getIblockId('CODE')`, `getHlblockIdIfExists('NAME')`.
- **Только схема и справочники** — без бизнес-логики и массового наполнения контентом.
- **Один логический блок — одна миграция**.

## Антипаттерны

- Создавать HL/инфоблок в админке без миграции.
- DDL в PHP-коде приложения или в `install/` модуля.
- Хардкод ID инфоблоков между окружениями.

## Чек-лист

- [ ] `up()` и `down()` симметричны.
- [ ] Использованы `save*` helpers.
- [ ] ID сущностей через код/XML_ID, не числовые константы.
- [ ] Миграция применена локально перед коммитом (если есть доступ к окружению).
