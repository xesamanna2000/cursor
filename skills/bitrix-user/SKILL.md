---
name: bitrix-user
description: Покрывает работу с пользователями в Bitrix — UserTable (D7), CUser (легаси), группы пользователей, авторизацию через $USER, проверку прав. Применяется при чтении/записи профиля пользователя, проверке принадлежности к группам, авторизации и разграничении доступа. Ключевые термины — UserTable, CUser, $USER, IsAuthorized, IsAdmin, группы пользователей.
---

# Пользователи в Bitrix

## D7 vs легаси

| Задача | D7 (предпочтительно) | Легаси / `$USER` |
|---|---|---|
| Чтение данных пользователя | `UserTable::getById()` / `getList()` | `CUser::GetByID()` |
| Список пользователей | `UserTable::getList()` с `limit` | `CUser::GetList()` |
| Создание пользователя (с паролем) | — | `CUser::Add()` |
| Обновление полей без пароля/групп | `UserTable::update()` | `CUser::Update()` |
| Смена пароля, группы, UF_* | — | `CUser::Update()` |
| Авторизация, сессия, текущий пользователь | — | `$USER` (`Login`, `Authorize`, `GetID`, …) |
| Проверка `IsAdmin` / групп | — | `$USER->IsAdmin()`, `IsInGroup()` |
| Группы произвольного пользователя | `UserGroupTable::getList()` | — |
| ORM-объект по ID | `UserTable::getByPrimary()->fetchObject()` | — |

Для **авторизации, сессии и текущего пользователя** — только глобальная `$USER`. Для **чтения профиля и списков** — D7 `UserTable`. Для **создания и паролей** — `CUser`.

## Текущий пользователь

```php
global $USER;

$userId = (int)$USER->GetID();       // ID текущего пользователя (0 если не авторизован)
$isAuth = $USER->IsAuthorized();     // авторизован ли
$isAdmin = $USER->IsAdmin();         // суперадмин
```

В контроллерах используй `ActionFilter\Authentication` для защиты действий:

```php
public function configureActions(): array
{
    return [
        'create' => [
            'prefilters' => [
                new \Bitrix\Main\Engine\ActionFilter\Authentication(),
                new \Bitrix\Main\Engine\ActionFilter\Csrf(),
            ],
        ],
    ];
}
```

## Чтение данных пользователя (D7)

```php
use Bitrix\Main\UserTable;

// По ID
$user = UserTable::getById(42)->fetch();
// ['ID' => 42, 'NAME' => 'Иван', 'LAST_NAME' => '...', 'EMAIL' => '...']

// Объектная модель
$userObject = UserTable::getByPrimary(42)->fetchObject();
echo $userObject->get('NAME');

// Список с фильтром
$users = UserTable::getList([
    'select' => ['ID', 'NAME', 'LAST_NAME', 'EMAIL', 'ACTIVE'],
    'filter' => ['=ACTIVE' => 'Y', '%EMAIL' => '@example.com'],
    'order'  => ['LAST_NAME' => 'ASC'],
    'limit'  => 50,
])->fetchAll();
```

Основные поля `UserTable`:

| Поле | Описание |
|---|---|
| `ID` | Идентификатор |
| `LOGIN` | Логин |
| `EMAIL` | Email |
| `NAME`, `LAST_NAME`, `SECOND_NAME` | Имя, фамилия, отчество |
| `ACTIVE` | Y/N |
| `GROUP_ID` | Массив ID групп (только через `CUser`) |
| `UF_*` | Пользовательские поля |
| `LAST_LOGIN` | Дата последнего входа |
| `DATE_REGISTER` | Дата регистрации |

## Создание и обновление пользователей

```php
// Создание с паролем — через CUser (корректное хеширование)
$cUser = new \CUser();
$userId = $cUser->Add([
    'LOGIN'            => 'ivan',
    'EMAIL'            => 'ivan@example.com',
    'NAME'             => 'Иван',
    'LAST_NAME'        => 'Иванов',
    'ACTIVE'           => 'Y',
    'PASSWORD'         => 'SecurePassword123',
    'CONFIRM_PASSWORD' => 'SecurePassword123',
]);

if (!$userId) {
    // $cUser->LAST_ERROR
}

// Обновление полей без пароля — D7
$result = UserTable::update($userId, [
    'NAME'      => 'Пётр',
    'LAST_NAME' => 'Петров',
]);

// Смена пароля и управление группами — только CUser
$cUser->Update($userId, [
    'PASSWORD'         => 'NewSecurePassword456',
    'CONFIRM_PASSWORD' => 'NewSecurePassword456',
    'GROUP_ID'         => [2, 5],
]);
```

`UserTable::add()` не подходит для создания пользователя с паролем — используй `CUser::Add()`.

## Группы пользователей

```php
// Проверить, состоит ли текущий пользователь в группе
global $USER;
$isInGroup = $USER->IsInGroup(5); // 5 — ID группы

// Группы текущего пользователя
$groupIds = $USER->GetUserGroupArray(); // [1, 2, 5]

// Группы произвольного пользователя (D7)
use Bitrix\Main\UserGroupTable;

$groups = UserGroupTable::getList([
    'filter' => ['=USER_ID' => 42],
    'select' => ['GROUP_ID'],
])->fetchAll();

$groupIds = array_column($groups, 'GROUP_ID');
```

Получить ID группы по коду:

```php
use Bitrix\Main\GroupTable;

$group = GroupTable::getList([
    'filter' => ['=STRING_ID' => 'MY_GROUP_CODE'],
    'select' => ['ID'],
])->fetch();

$groupId = $group['ID'] ?? null;
```

## Авторизация

```php
global $USER;

// Логин по логину и паролю
$result = $USER->Login('ivan', 'password', 'Y'); // 'Y' — запомнить
if ($result !== true) {
    // $result — строка с ошибкой
}

// Логин без пароля (административная авторизация)
$USER->Authorize($userId);

// Выход
$USER->Logout();
```

## Пользовательские поля (UF_*)

Поля `UF_*` пользователя получаются через `CUser`:

```php
$userFields = \CUser::GetByID($userId)->GetNext();
$phone = $userFields['UF_PHONE'] ?? '';
```

Через D7 — только если поля явно добавлены в select и проиндексированы:

```php
$row = UserTable::getList([
    'filter' => ['=ID' => $userId],
    'select' => ['ID', 'UF_PHONE', 'UF_DEPARTMENT'],
])->fetch();
```

Обновление пользовательских полей:

```php
$cUser = new \CUser();
$cUser->Update($userId, ['UF_PHONE' => '+79001234567']);
```

## Права и доступ (Access)

Для гранулярного контроля доступа используй `\Bitrix\Main\Access`:

```php
global $USER;

// Проверка конкретного права (модульные сущности с AccessibleItem)
$canEdit = \Bitrix\Main\Access\AccessController::can(
    (int)$USER->GetID(),
    \Bitrix\Main\Access\ActionDictionary::ACTION_ELEMENT_EDIT,
    $item, // объект, реализующий AccessibleItem
);
```

Для простых случаев — проверка группы или `IsAdmin()`:

```php
if (!$USER->IsAdmin() && !$USER->IsInGroup($editorGroupId)) {
    throw new \Bitrix\Main\AccessDeniedException();
}
```

## Антипаттерны

- Не обращаться к `$_SESSION['SESS_AUTH']` напрямую — только через `$USER`.
- Не делать `new CUser()` для чтения данных — используй D7 `UserTable`.
- Не хардкодить ID групп — получать через `GroupTable` по `STRING_ID`.
- Не использовать `CUser::GetList()` для больших выборок — только D7 `UserTable::getList()` с `limit`.
- Не забывать `global $USER` — это глобальная переменная Bitrix, не инжектируемый сервис.
