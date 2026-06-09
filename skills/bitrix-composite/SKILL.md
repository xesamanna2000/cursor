---
name: bitrix-composite
description: Покрывает композитный режим Bitrix (Composite Site) — кеш страниц, динамические области через createFrame и FrameStatic, startDynamicWithID вне компонентов, отключение composite, диагностика. Применяется при разработке компонентов и страниц на сайтах с включённым композитным режимом. Ключевые термины — composite, createFrame, FrameStatic, FrameBuffered, startDynamicWithID, BX.ajax, Nginx page cache.
---

# Композитный режим (Composite Site)

Документация: [урок по динамическим областям](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=39&LESSON_ID=2552), [Composite Site](https://docs.1c-bitrix.ru/pages/performance/composite-site.html).

## Как работает

Composite — двухуровневый кеш:

1. **Статическая часть страницы** записывается в файловый кеш (`/bitrix/html_pages/`) и может отдаваться без полного прогона PHP.
2. **Динамические области** (корзина, авторизация, персональные данные) вырезаются из кеша и дозагружаются отдельным AJAX-запросом после отдачи страницы.

Всё, что не обёрнуто в динамическую область, попадает в общий кеш страницы.

## Динамическая область в шаблоне компонента

Основной способ — `$this->createFrame()` в `template.php`. Сначала рендерится динамическая часть, затем заглушка (`beginStub`), которая показывается до AJAX-дозагрузки:

```php
<?php if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) { die(); }

$frame = $this->createFrame('user-panel')->begin('Загрузка...');
?>
    <?php global $USER; ?>
    <?php if ($USER->IsAuthorized()): ?>
        <span><?= htmlspecialcharsbx($USER->GetFullName()) ?></span>
    <?php else: ?>
        <a href="/login/">Войти</a>
    <?php endif; ?>
<?php
$frame->beginStub();
?>
    <span>Загрузка...</span>
<?php
$frame->end();
```

Опции фрейма:

```php
$frame = $this->createFrame('price-block', false)->begin('');
$frame->setAnimation(true);       // плавное появление после дозагрузки
$frame->setBrowserStorage(true); // кеш динамики в localStorage браузера
```

Частичное кеширование внутри компонента (например, цена):

```php
$frame = $this->createFrame('price-index', false)->begin();
echo $arResult['PRICE'];
$frame->beginStub();
echo 'руб.';
$frame->end();
```

## Динамическая область вне компонента

В `header.php`, `footer.php` или других файлах шаблона сайта — `FrameStatic` или синглтон `Frame`:

```php
// FrameStatic — без буферизации, расставляет метки начала/конца зоны
$dynamicArea = new \Bitrix\Main\Page\FrameStatic('top-banner');
$dynamicArea->setAnimation(true);
$dynamicArea->setStub('Загрузка...');
$dynamicArea->setContainerID('top-banner-container');
$dynamicArea->startDynamicArea();
// HTML, PHP, IncludeComponent
$dynamicArea->finishDynamicArea();
```

Альтернатива через синглтон:

```php
\Bitrix\Main\Page\Frame::getInstance()->startDynamicWithID('header-cart');
// динамический контент
\Bitrix\Main\Page\Frame::getInstance()->finishDynamicWithID('header-cart', 'Загрузка...');
```

## FrameBuffered вне шаблона компонента

Если нужна буферизация с заглушкой без `$this`:

```php
$frame = new \Bitrix\Main\Page\FrameBuffered('my_dynamic');
$frame->begin();
    // динамический контент
$frame->beginStub();
    // заглушка
$frame->end();
```

## Отключение composite

Для всей страницы:

```php
\Bitrix\Main\Composite\Engine::disableCompositeForPage();
```

В `class.php` компонента, если нельзя разделить вывод на статику/динамику:

```php
public function executeComponent(): void
{
    if (\Bitrix\Main\Composite\Engine::isEnabled()) {
        \Bitrix\Main\Composite\Engine::disableCompositeForPage();
    }
    // ...
}
```

Отключить автообновление динамической зоны:

```php
\Bitrix\Main\Page\Frame::getInstance()->setAutoUpdate(false);
\Bitrix\Main\Page\Frame::getInstance()->setAutoUpdateTTL(60);
```

## Что ломает composite

| Проблема | Причина | Решение |
|---|---|---|
| Страница не кешируется | `die()` / `exit()` до конца рендера | Убрать отладочные остановки, использовать `HttpResponse` |
| Чужие данные в кеше | `$USER->GetID()` вне динамической области | Обернуть в `createFrame()` / `FrameStatic` |
| CSRF-токен протух | `bitrix_sessid()` в статическом HTML | Добавлять через JS: `BX.bitrix_sessid()` |
| Кеш не инвалидируется | Нет тегов при изменении данных | `ManagedCache` + тег по таблице ORM (см. `bitrix-caching`) |
| Корзина/счётчики не обновляются | Блок в статической зоне | Вынести в динамический фрейм или `BX.ajax.runAction` |

## Диагностика

Заголовок ответа `X-Bitrix-Composite`: `hit` — из кеша, `miss` — PHP.

Сброс файлового кеша:

```bash
find /path/to/site/bitrix/html_pages/ -name "*.html" -delete
```

Проверка, можно ли кешировать текущую страницу:

```php
$compositeEnabled = \Bitrix\Main\Composite\Engine::isEnabled()
    && \Bitrix\Main\Composite\Engine::isPageCacheable();
```

## Правила работы с composite

- **Не выводить данные текущего пользователя вне динамической области** — они попадут в кеш и будут показаны другим.
- **Не использовать `bitrix_sessid_post()`** в статических формах — CSRF-токен протухнет. Добавлять через JS: `BX.bitrix_sessid()`.
- **Не использовать `die()`/`exit()`** вне стандартных guard'ов — прерывают запись кеша.
- **Тестировать в режиме инкогнито** — браузерная сессия не должна влиять на проверку Nginx-кеша.

## Антипаттерны

- Не использовать несуществующие API (`defineFrameWithHeader`, `Composite\Frame`) — только `createFrame`, `FrameStatic`, `FrameBuffered`, `Frame::getInstance()`.
- Не путать `define('NO_AGENT_STATISTIC', 'Y')` с отключением composite — это про агентов статистики.
- Не кешировать персональный контент «надеясь, что composite сам разберётся» — явно оборачивать в динамическую область.
