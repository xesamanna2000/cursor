---
name: bitrix-admin
description: Покрывает страницы административного раздела Bitrix — список (CAdminList), форма редактирования (CAdminForm, CAdminTabControl), регистрация пунктов меню, права доступа в админке, admin-компоненты. Применяется при создании UI модуля в административном разделе. Ключевые термины — CAdminList, CAdminForm, CAdminTabControl, menu.php, admin page, IsAdmin.
---

# Административный раздел Bitrix

> **Правило `no-module-install`:** страницы в `admin/` модуля редактируем; регистрацию меню через `DoInstall` в `install/` — не предлагаем. Меню настраивается вручную или механизмом проекта.

## Структура файлов

Страницы админки модуля живут в `/local/modules/<vendor>.<module>/`:

```
vendor.module/
├── install/
│   └── index.php          # регистрирует пункты меню через DoInstall
├── admin/
│   ├── vendor_module_list.php    # страница списка
│   └── vendor_module_edit.php    # страница редактирования
└── lang/ru/admin/
    ├── vendor_module_list.php
    └── vendor_module_edit.php
```

На проекте страницы лежат в `/local/modules/vendor.module/admin/` и подключаются через меню вручную или механизмом проекта. Справочно: при `DoInstall` файлы могут копироваться в `/bitrix/admin/`.

## Регистрация пункта меню (справочно)

В стандартном Bitrix — в `DoInstall`. **На проекте не предлагаем** — меню настраивается вручную:

```php
\CAdminMenu::AddSubMenu(
    'global_menu_content',                    // раздел главного меню
    'vendor_module',                          // ID подменю
    Loc::getMessage('VENDOR_MODULE_MENU'),    // заголовок
    [
        [
            'text'  => Loc::getMessage('VENDOR_MODULE_LIST'),
            'url'   => 'vendor_module_list.php?lang=' . LANGUAGE_ID,
            'more_url' => ['vendor_module_edit.php'],
            'icon'  => 'fileman_menu_icon',
        ],
    ],
    $sort = 100,
);
```

Справочно, снятие при `DoUninstall`: `\CAdminMenu::RemoveSubMenu('global_menu_content', 'vendor_module');`

## Страница списка

```php
<?php
// /local/modules/vendor.module/admin/vendor_module_list.php
require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_admin_before.php';

use Bitrix\Main\Localization\Loc;
use Bitrix\Main\Loader;
use Vendor\Module\Model\PostTable;

Loader::includeModule('vendor.module');
Loc::loadMessages(__FILE__);

// Проверка прав
global $USER, $APPLICATION;
if (!$USER->IsAdmin()) {
    $APPLICATION->AuthForm(Loc::getMessage('ACCESS_DENIED'));
}

// Обработка действий (удаление, активация и т.д.)
// check_bitrix_sessid() — правильный способ CSRF-защиты для admin-страниц.
// Это не то же самое, что ActionFilter\Csrf в D7-контроллерах — у admin-страниц своя архитектура.
$request = \Bitrix\Main\Application::getInstance()->getContext()->getRequest();
if ($request->isPost() && check_bitrix_sessid()) {
    $action = $request->getPost('action');
    $ids    = (array)$request->getPost('ID');

    if ($action === 'delete') {
        foreach ($ids as $id) {
            PostTable::delete((int)$id);
        }
    }
}

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_admin_after.php';

// --- Фильтр ---
$filterFields = [
    ['id' => 'TITLE', 'name' => Loc::getMessage('VENDOR_MODULE_TITLE'), 'type' => 'string'],
    ['id' => 'ACTIVE', 'name' => Loc::getMessage('VENDOR_MODULE_ACTIVE'), 'type' => 'list',
     'items' => ['' => '...', 'Y' => Loc::getMessage('YES'), 'N' => Loc::getMessage('NO')]],
];

$adminFilter = new CAdminUiFilter('vendor_module_list_filter', $filterFields);
$adminFilter->InitFilter($filterValues);

$filter = [];
if (!empty($filterValues['TITLE'])) {
    $filter['%TITLE'] = $filterValues['TITLE'];
}
if (!empty($filterValues['ACTIVE'])) {
    $filter['=ACTIVE'] = $filterValues['ACTIVE'];
}

// --- Сортировка ---
$by    = $request->get('by') ?? 'ID';
$order = $request->get('order') ?? 'desc';
$allowedBy = ['ID', 'TITLE', 'ACTIVE'];
if (!in_array(strtoupper($by), $allowedBy, true)) {
    $by = 'ID';
}

// --- Данные ---
$dbResult = PostTable::getList([
    'select' => ['ID', 'TITLE', 'ACTIVE', 'CREATED_AT'],
    'filter' => $filter,
    'order'  => [strtoupper($by) => strtoupper($order) === 'ASC' ? 'ASC' : 'DESC'],
]);

// --- Список ---
$adminList = new CAdminList('vendor_module_list', new CAdminSorting(
    'vendor_module_list', 'ID', 'desc'
));

$adminList->AddHeaders([
    ['id' => 'ID',         'content' => 'ID',                           'sort' => 'ID',     'default' => true],
    ['id' => 'TITLE',      'content' => Loc::getMessage('VENDOR_MODULE_TITLE'),  'sort' => 'TITLE',  'default' => true],
    ['id' => 'ACTIVE',     'content' => Loc::getMessage('VENDOR_MODULE_ACTIVE'), 'sort' => 'ACTIVE', 'default' => true],
    ['id' => 'CREATED_AT', 'content' => Loc::getMessage('VENDOR_MODULE_DATE'),   'default' => true],
]);

while ($row = $dbResult->fetch()) {
    $adminRow = $adminList->AddRow($row['ID'], $row);
    $adminRow->AddViewField('ACTIVE', $row['ACTIVE'] === 'Y'
        ? Loc::getMessage('YES') : Loc::getMessage('NO'));
    $adminRow->AddActions([
        ['text' => Loc::getMessage('EDIT'),   'action' => "document.location='vendor_module_edit.php?ID={$row['ID']}&lang=" . LANGUAGE_ID . "'"],
        ['text' => Loc::getMessage('DELETE'), 'action' => "if(confirm('" . Loc::getMessage('CONFIRM_DELETE') . "')) document.location='vendor_module_list.php?action=delete&ID={$row['ID']}&lang=" . LANGUAGE_ID . "&" . bitrix_sessid_get() . "'", 'icon' => 'delete'],
    ]);
}

$adminList->AddGroupActionTable([
    'delete' => Loc::getMessage('DELETE'),
]);

$adminList->CheckListMode();
$adminFilter->ShowFilter();
$adminList->DisplayList();

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/epilog_admin.php';
```

## Страница редактирования (форма с табами)

```php
<?php
// /local/modules/vendor.module/admin/vendor_module_edit.php
require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_admin_before.php';

use Bitrix\Main\Loader;
use Bitrix\Main\Localization\Loc;
use Vendor\Module\Model\PostTable;

Loader::includeModule('vendor.module');
Loc::loadMessages(__FILE__);

global $USER, $APPLICATION;
if (!$USER->IsAdmin()) {
    $APPLICATION->AuthForm(Loc::getMessage('ACCESS_DENIED'));
}

$request = \Bitrix\Main\Application::getInstance()->getContext()->getRequest();
$id      = (int)$request->get('ID');
$isNew   = $id <= 0;

$fields = [];
$errors = [];

// Загрузка существующей записи
if (!$isNew) {
    $fields = PostTable::getById($id)->fetch();
    if (!$fields) {
        $APPLICATION->ThrowException(Loc::getMessage('RECORD_NOT_FOUND'));
    }
}

// Сохранение
if ($request->isPost() && check_bitrix_sessid()) {
    $fields = [
        'TITLE'  => trim((string)$request->getPost('TITLE')),
        'ACTIVE' => $request->getPost('ACTIVE') === 'Y' ? 'Y' : 'N',
    ];

    $result = $isNew
        ? PostTable::add($fields)
        : PostTable::update($id, $fields);

    if ($result->isSuccess()) {
        LocalRedirect('vendor_module_list.php?lang=' . LANGUAGE_ID);
    } else {
        $errors = $result->getErrorMessages();
    }
}

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_admin_after.php';

// --- Форма ---
$tabControl = new CAdminTabControl('tabControl', [
    ['DIV' => 'edit1', 'TAB' => Loc::getMessage('TAB_MAIN'), 'ICON' => 'main_user_edit', 'TITLE' => Loc::getMessage('TAB_MAIN_TITLE')],
]);

$tabControl->Begin();
?>
<form method="post" action="vendor_module_edit.php?lang=<?= LANGUAGE_ID ?>">
    <?= bitrix_sessid_post() ?>
    <?php if ($id > 0): ?><input type="hidden" name="ID" value="<?= $id ?>"><?php endif; ?>

    <?php $tabControl->BeginNextTab(); ?>

    <?php foreach ($errors as $error): ?>
        <div class="adm-info-message-wrap adm-info-message-red">
            <div class="adm-info-message"><?= htmlspecialchars($error) ?></div>
        </div>
    <?php endforeach; ?>

    <tr>
        <td width="40%"><?= Loc::getMessage('VENDOR_MODULE_TITLE') ?>:</td>
        <td><input type="text" name="TITLE" value="<?= htmlspecialchars((string)($fields['TITLE'] ?? '')) ?>" size="50"></td>
    </tr>
    <tr>
        <td><?= Loc::getMessage('VENDOR_MODULE_ACTIVE') ?>:</td>
        <td><input type="checkbox" name="ACTIVE" value="Y"<?= ($fields['ACTIVE'] ?? '') === 'Y' ? ' checked' : '' ?>></td>
    </tr>

    <?php $tabControl->Buttons(['back_url' => 'vendor_module_list.php?lang=' . LANGUAGE_ID]); ?>
</form>
<?php
$tabControl->End();
require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/epilog_admin.php';
```

## Проверка прав в admin-страницах

```php
global $USER, $APPLICATION;

// Только суперадмин
if (!$USER->IsAdmin()) {
    $APPLICATION->AuthForm(Loc::getMessage('ACCESS_DENIED'));
}

// Проверка модульных прав (D < R < W)
if (\CMain::GetGroupRight('vendor.module') < 'W') {
    $APPLICATION->AuthForm(Loc::getMessage('ACCESS_DENIED'));
}
```

## Антипаттерны

- Не подключать `prolog_admin_after.php` до обработки POST — логика сохранения должна быть **до** подключения пролога.
- Не забывать `check_bitrix_sessid()` на все POST-запросы в админке.
- Не выводить данные без `htmlspecialchars()` в шаблонах форм.
- Не использовать устаревший `$APPLICATION->IncludeAdminFile()` для сложных страниц.
- Не делать прямые SQL-запросы в admin-файлах — только через ORM и сервисы.
