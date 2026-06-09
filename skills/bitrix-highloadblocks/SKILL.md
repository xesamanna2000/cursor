---
name: bitrix-highloadblocks
description: Покрывает High-Load блоки (HL-блоки) Bitrix — HighloadBlockTable, compileEntity, динамический ORM-класс, поля UF_*, HighloadBlockRightsTable, чтение/запись, кеш. Применяется при работе с HL-блоками как ORM-таблицами вместо инфоблоков для высоконагруженных справочников. Ключевые термины — HighloadBlockTable, compileEntity, HighloadBlockRightsTable, hlblock, UF_*.
---

# High-Load блоки (HL-блоки)

> **Правило `no-module-install`:** подписку на события не оформляем в `install/index.php` — используй `init.php` или bootstrap проекта.

HL-блоки — это ORM-таблицы без привязки к инфоблочной структуре. Используются для справочников, пользовательских полей и данных с высокой частотой чтения/записи.

Перед работой с HL-блоками подключи модуль:

```php
\Bitrix\Main\Loader::includeModule('highloadblock');
```

## Получение ORM-класса HL-блока

HL-блок не имеет фиксированного PHP-класса — он компилируется динамически по имени или ID.

```php
use Bitrix\Highloadblock\HighloadBlockTable;
use Bitrix\Main\Loader;

Loader::includeModule('highloadblock');

// По имени HL-блока
$hlblock = HighloadBlockTable::getList([
    'filter' => ['=NAME' => 'Color'],
])->fetch();

$entity = HighloadBlockTable::compileEntity($hlblock);
$dataClass = $entity->getDataClass(); // динамический класс, например Bitrix\Hlblock\ColorTable
```

Для удобства — вспомогательный метод:

```php
function getHlClass(string $name): string
{
    Loader::includeModule('highloadblock');
    $hlblock = HighloadBlockTable::getList(['filter' => ['=NAME' => $name]])->fetch();
    if (!$hlblock) {
        throw new \RuntimeException("HL-block '{$name}' not found");
    }
    return HighloadBlockTable::compileEntity($hlblock)->getDataClass();
}

$colorClass = getHlClass('Color');
```

## Чтение данных

После получения `$dataClass` работа идёт через стандартный ORM-интерфейс (см. `bitrix-orm`):

```php
$rows = $colorClass::getList([
    'select' => ['ID', 'UF_NAME', 'UF_XML_ID', 'UF_SORT'],
    'filter' => ['=UF_ACTIVE' => '1'],
    'order'  => ['UF_SORT' => 'ASC'],
])->fetchAll();

// Объектная модель
$color = $colorClass::getByPrimary(5)->fetchObject();
echo $color->get('UF_NAME');
```

## Запись данных

```php
// Добавление
$result = $colorClass::add([
    'UF_NAME'   => 'Красный',
    'UF_XML_ID' => 'red',
    'UF_SORT'   => 10,
    'UF_ACTIVE' => '1',
]);

if (!$result->isSuccess()) {
    // обработка ошибок через $result->getErrorMessages()
}

$newId = $result->getId();

// Обновление
$colorClass::update($newId, ['UF_SORT' => 20]);

// Удаление
$colorClass::delete($newId);
```

## Поля HL-блока

Поля HL-блока — это пользовательские поля (`UF_*`), управляемые через `CUserTypeManager` / `\Bitrix\Main\UserField\Types`.

Все поля начинаются с `UF_`. Тип задаётся при создании:

| Тип | Код типа |
|---|---|
| Строка | `string` |
| Число | `integer`, `double` |
| Дата/Время | `datetime`, `date` |
| Список (enum) | `enumeration` |
| Файл | `file` |
| Привязка к элементу HL | `hlblock` |
| Привязка к элементу инфоблока | `iblock_element` |
| Чекбокс | `boolean` |

Создание поля вручную (обычно делается через sprint.migration):

```php
$userTypeManager = new \CUserTypeManager();
$userTypeManager->Add([
    'ENTITY_ID'        => 'HLBLOCK_' . $hlblock['ID'],
    'FIELD_NAME'       => 'UF_NAME',
    'USER_TYPE_ID'     => 'string',
    'XML_ID'           => 'UF_NAME',
    'SORT'             => 100,
    'MULTIPLE'         => 'N',
    'MANDATORY'        => 'Y',
    'EDIT_FORM_LABEL'  => ['ru' => 'Название', 'en' => 'Name'],
    'LIST_COLUMN_LABEL'=> ['ru' => 'Название', 'en' => 'Name'],
]);
```

## Создание HL-блока

Через `sprint.migration` (см. скилл `bitrix-sprint-migration`):

```php
// В миграции:
$helper->Hlblock()->saveHlblock([
    'NAME'       => 'Color',
    'TABLE_NAME' => 'b_hlblock_color',
]);

$helper->Hlblock()->saveField('Color', [
    'FIELD_NAME'   => 'UF_NAME',
    'USER_TYPE_ID' => 'string',
    'MANDATORY'    => 'Y',
    'SORT'         => 100,
]);
```

Вручную (если миграций нет):

```php
$result = HighloadBlockTable::add([
    'NAME'       => 'Color',
    'TABLE_NAME' => 'b_hlblock_color',
]);
$hlblockId = $result->getId();
```

## Права доступа

Проверка прав на уровне запросов ORM **не встроена** — проверяй вручную перед операциями:

```php
use Bitrix\Highloadblock\HighloadBlockRightsTable;
use Bitrix\Highloadblock\HighloadBlockTable;

$hlblock = HighloadBlockTable::getList([
    'filter' => ['=NAME' => 'Color'],
    'select' => ['ID'],
])->fetch();

$hlId = (int)$hlblock['ID'];
$operations = HighloadBlockRightsTable::getOperationsName($hlId);
// массив операций для текущего пользователя, например ['hlblock_read', 'hlblock_write']

if (!in_array('hlblock_write', $operations ?? [], true)) {
    throw new \Bitrix\Main\AccessDeniedException();
}
```

Назначение прав — через sprint.migration (`$helper->Hlblock()->saveGroupPermissions(...)`) или админку с фиксацией в миграции.

## События ORM

HL-блок поддерживает стандартные ORM-события:

```php
$entity->addEventHandler('onBeforeAdd', function (\Bitrix\Main\ORM\Event $event) {
    $fields = $event->getParameter('fields');
    // валидация или модификация полей
});
```

Глобальная подписка через `EventManager` (на проекте — в `init.php`; справочно в Bitrix также `install/index.php`):

```php
\Bitrix\Main\EventManager::getInstance()->addEventHandler(
    'main',
    'OnBeforeUserTypeEntityUpdate',
    [\Vendor\Module\Internals\Integration\Main\EventHandler\HlblockHandler::class, 'handle'],
);
```

## Кеширование

HL-блок через ORM кешируется стандартно — тег тега кеша равен имени таблицы:

```php
use Bitrix\Main\Data\Cache;

$cache = Cache::createInstance();
if ($cache->initCache(3600, 'hlblock_color_list', '/hlblock/color')) {
    $data = $cache->getVars();
} elseif ($cache->startDataCache()) {
    $data = $colorClass::getList(['select' => ['ID', 'UF_NAME', 'UF_XML_ID']])->fetchAll();
    $cache->endDataCache($data);
}
```

## Антипаттерны

- Не хардкодить ID HL-блока — он отличается на разных окружениях. Всегда получать через `NAME`.
- Не использовать `$_GET`/`$_POST` для подстановки в `filter` без белого списка — риск инъекции через ORM.
- Не создавать HL-блок и его поля напрямую в PHP-скрипте на боевом сервере — только через миграцию.
- Не вызывать `compileEntity` в цикле — кешировать `$dataClass` в переменную.
