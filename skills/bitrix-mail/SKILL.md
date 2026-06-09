---
name: bitrix-mail
description: Покрывает отправку почты в Bitrix — \Bitrix\Main\Mail\Event::send() и устаревший CEvent::Send(), создание типов почтовых событий и шаблонов писем через sprint.migration, плейсхолдеры, вложения, отладка. Применяется при отправке транзакционных писем: регистрация, уведомления, подтверждения. Ключевые термины — Mail\Event, CEvent, почтовое событие, шаблон письма, EventMessageTable.
---

# Отправка почты в Bitrix

## Основной способ: `\Bitrix\Main\Mail\Event::send()`

```php
use Bitrix\Main\Mail\Event;
use Bitrix\Main\UserTable;

$user = UserTable::getById($userId)->fetch();

$result = Event::send([
    'EVENT_NAME' => 'VENDOR_MODULE_NEW_ORDER',
    'LID'        => SITE_ID,
    'C_FIELDS'   => [
        'ORDER_ID'   => $order->getId(),
        'USER_NAME'  => trim(($user['NAME'] ?? '') . ' ' . ($user['LAST_NAME'] ?? '')),
        'USER_EMAIL' => $user['EMAIL'] ?? '',
        'ORDER_LINK' => 'https://site.ru/orders/' . $order->getId() . '/',
    ],
]);

if (!$result->isSuccess()) {
    // $result->getErrorMessages()
}
```

`Event::send()` ищет активный шаблон по коду события и сайту, подставляет плейсхолдеры и отправляет письмо через настроенный SMTP.

## Создание типа события и шаблона

Через sprint.migration (обязательно — см. правило `sprint-migration`):

```php
public function up(): void
{
    $helper = $this->getHelperManager();

    // Тип почтового события
    $helper->Event()->saveEventType([
        'LID'          => 'ru',
        'EVENT_NAME'   => 'VENDOR_MODULE_NEW_ORDER',
        'NAME'         => 'Новый заказ — уведомление менеджеру',
        'DESCRIPTION'  => "ORDER_ID - ID заказа\nUSER_NAME - Имя клиента\nUSER_EMAIL - Email клиента\nORDER_LINK - Ссылка на заказ",
    ]);

    // Шаблон письма
    $helper->Event()->saveEventMessage([
        'EVENT_NAME'  => 'VENDOR_MODULE_NEW_ORDER',
        'LID'         => ['ru'],
        'ACTIVE'      => 'Y',
        'EMAIL_FROM'  => '#DEFAULT_EMAIL_FROM#',
        'EMAIL_TO'    => 'manager@site.ru',
        'SUBJECT'     => 'Новый заказ #ORDER_ID#',
        'BODY_TYPE'   => 'html',
        'MESSAGE'     => '<p>Поступил новый заказ <a href="#ORDER_LINK#">#ORDER_ID#</a></p>
<p>Клиент: #USER_NAME# (#USER_EMAIL#)</p>',
    ]);
}

public function down(): void
{
    $helper = $this->getHelperManager();
    $helper->Event()->deleteEventTypeIfExists('VENDOR_MODULE_NEW_ORDER');
}
```

## Отправка с явным указанием получателя

Если шаблон не настроен или нужно динамически переопределить получателя:

```php
Event::send([
    'EVENT_NAME' => 'VENDOR_MODULE_NEW_ORDER',
    'LID'        => SITE_ID,
    'C_FIELDS'   => [
        'ORDER_ID'  => 42,
        'USER_NAME' => 'Иван',
    ],
    'TO_EMAIL'   => 'custom@example.com',  // переопределить получателя
]);
```

## Письмо с вложением

```php
use Bitrix\Main\Mail\Event;

Event::send([
    'EVENT_NAME' => 'VENDOR_MODULE_REPORT',
    'LID'        => SITE_ID,
    'C_FIELDS'   => ['REPORT_NAME' => 'Отчёт за июнь'],
    'FILE'       => [
        [
            'FILE_NAME'    => 'report.pdf',
            'CONTENT_TYPE' => 'application/pdf',
            'CONTENT'      => file_get_contents('/path/to/report.pdf'),
        ],
    ],
]);
```

## Системные плейсхолдеры

Доступны в шаблонах без передачи в `C_FIELDS`:

| Плейсхолдер | Значение |
|---|---|
| `#DEFAULT_EMAIL_FROM#` | Email отправителя из настроек сайта |
| `#SITE_NAME#` | Название сайта |
| `#SERVER_NAME#` | Домен сайта |

## Прямая отправка без шаблона (не рекомендуется)

Только если событие и шаблон избыточны (например, служебные уведомления разработчику):

```php
use Bitrix\Main\Mail\Mail;

Mail::send([
    'SUBJECT'      => 'Ошибка импорта',
    'BODY'         => 'Детали: ' . $message,
    'EMAIL_FROM'   => 'noreply@site.ru',
    'EMAIL_TO'     => 'dev@site.ru',
    'CONTENT_TYPE' => 'text',
]);
```

## Отладка: почему письмо не отправляется

1. Проверить, есть ли активный шаблон для события и сайта:

```php
use Bitrix\Main\Mail\EventMessageTable;

$template = EventMessageTable::getList([
    'filter' => ['=EVENT_NAME' => 'VENDOR_MODULE_NEW_ORDER', '=ACTIVE' => 'Y'],
])->fetch();

// $template === false — шаблон не найден или не активен для EVENT_NAME и сайта
// Для логирования в проде — PSR-3 логгер (см. bitrix-logger), не var_dump
```

2. Включить логирование почты в `/local/php_interface/init.php`:

```php
define('BX_FILE_PERMISSIONS', 0664);
// Логи писем: /bitrix/modules/main/classes/general/event.php пишет в mail_log
```

3. Проверить очередь отправки в БД: таблица `b_event`.

## Антипаттерны

- Не использовать `mail()` или `CEvent::Send()` напрямую — только `\Bitrix\Main\Mail\Event::send()`.
- Не хардкодить `EMAIL_TO` в коде — адреса должны быть в шаблоне или настройках модуля.
- Не создавать типы событий и шаблоны вручную через админку без фиксации в миграции.
- Не передавать HTML в `C_FIELDS` без экранирования — плейсхолдеры подставляются как есть.
