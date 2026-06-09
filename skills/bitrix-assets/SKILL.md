---
name: bitrix-assets
description: Покрывает подключение JS и CSS в Bitrix — Asset, UI Extension::load, CJSCore, скрипты в компонентах и шаблонах. Применяется при добавлении стилей, скриптов, Bitrix UI-расширений и управлении порядком загрузки. Ключевые термины — Asset, Extension::load, CJSCore, addJs, addCss, addString.
---

# Подключение JS и CSS (Assets)

## Asset — основной способ

```php
use Bitrix\Main\Page\Asset;

$asset = Asset::getInstance();
$asset->addJs('/local/js/vendor/my-script.js');
$asset->addCss('/local/templates/.default/css/custom.css');
$asset->addString('<meta name="viewport" content="width=device-width, initial-scale=1">');
```

Пути — от корня сайта (`/local/...`). В компоненте — в `executeComponent()` или `component_epilog.php`, не в `template.php` с тяжёлой логикой.

## Bitrix UI-расширения

```php
\Bitrix\Main\UI\Extension::load('ui.notification');
\Bitrix\Main\UI\Extension::load(['ui.buttons', 'ui.alerts']);
```

Расширения подтягивают JS/CSS пакета из ядра. Предпочтительнее для стандартных UI-виджетов.

## CJSCore (легаси, но ещё встречается)

```php
\CJSCore::Init(['ajax', 'popup']);
```

Для нового кода — `Extension::load` и `Asset`; `CJSCore` — если компонент/шаблон уже на нём или нужен конкретный legacy-пакет.

## В шаблоне компонента

```php
// template.php — только подключение уже зарегистрированных ассетов или inline с BX.message
\Bitrix\Main\Page\Asset::getInstance()->addJs($templateFolder . '/script.js');
\Bitrix\Main\Page\Asset::getInstance()->addCss($templateFolder . '/style.css');
```

`$templateFolder` — путь к текущему шаблону компонента.

## В class.php компонента

```php
public function onPrepareComponentParams($arParams): array
{
    \Bitrix\Main\UI\Extension::load('ui.dialogs.messagebox');
    return $arParams;
}
```

Или в `executeComponent()` перед `includeComponentTemplate()`.

## Порядок и defer

```php
Asset::getInstance()->addJs('/local/js/app.js', true);  // второй аргумент — в <head> vs перед </body> (зависит от версии)
```

Для критичных скриптов страницы — регистрируй в `header.php` шаблона сайта или в `component_epilog.php`.

## Связь с bitrix-js

Клиентская логика — скилл `bitrix-js` (`BX.ready`, `BX.ajax.runAction`). Ассеты только **подключают** файлы; инициализация — в `script.js` шаблона.

## Антипаттерны

- Не вставлять `<script src="...">` вручную в `template.php` — используй `Asset::addJs`.
- Не подключать jQuery напрямую — `BX` и `Extension::load`.
- Не дублировать один и тот же `addJs` на каждом хите без кеша компонента — выноси в шаблон сайта, если скрипт глобальный.
- Не класть бизнес-логику в `script.js` шаблона — только UI; API — через `BX.ajax.runAction`.
