---
name: bitrix-seo
description: Покрывает SEO-инструменты Bitrix — установка title/description/keywords через $APPLICATION, OpenGraph и мета-теги, canonical URL, SEO-шаблоны инфоблоков, robots и sitemap, работа с \Bitrix\Main\Seo\MetaTemplate. Применяется при настройке мета-данных страниц, SEO-шаблонов разделов и элементов инфоблоков, генерации sitemap. Ключевые термины — SetTitle, SetPageProperty, OG-теги, canonical, MetaTemplate, sitemap, robots.
---

# SEO в Bitrix

## Мета-данные страницы

Устанавливаются через глобальный объект `$APPLICATION`. Вызывать до `require prolog_before.php` или в компоненте до вывода шаблона.

```php
global $APPLICATION;

// Заголовок страницы (<title>)
$APPLICATION->SetTitle('Каталог товаров');

// Мета-теги
$APPLICATION->SetPageProperty('description', 'Описание страницы для поисковиков, до 160 символов.');
$APPLICATION->SetPageProperty('keywords', 'ключевое слово, ещё одно');

// Canonical URL (предотвращает дублирование)
$APPLICATION->SetPageProperty('canonical', 'https://site.ru/catalog/');

// robots
$APPLICATION->SetPageProperty('robots', 'noindex, nofollow');  // запрет индексации
```

В шаблоне сайта значения выводятся через:

```php
// header.php шаблона
<title><?= $APPLICATION->GetTitle() ?></title>
<meta name="description" content="<?= htmlspecialchars($APPLICATION->GetPageProperty('description')) ?>">
<meta name="keywords"    content="<?= htmlspecialchars($APPLICATION->GetPageProperty('keywords')) ?>">
<?php if ($canonical = $APPLICATION->GetPageProperty('canonical')): ?>
<link rel="canonical" href="<?= htmlspecialchars($canonical) ?>">
<?php endif; ?>
```

## OpenGraph-теги

```php
global $APPLICATION;

$APPLICATION->SetPageProperty('og:title',       'Заголовок для соцсетей');
$APPLICATION->SetPageProperty('og:description', 'Описание для соцсетей');
$APPLICATION->SetPageProperty('og:image',       'https://site.ru/upload/og-image.jpg');
$APPLICATION->SetPageProperty('og:type',        'article');
$APPLICATION->SetPageProperty('og:url',         'https://site.ru/news/post-slug/');
```

В шаблоне сайта:

```php
<?php
$ogProperties = ['og:title', 'og:description', 'og:image', 'og:type', 'og:url'];
foreach ($ogProperties as $prop):
    $value = $APPLICATION->GetPageProperty($prop);
    if ($value):
?>
<meta property="<?= htmlspecialchars($prop) ?>" content="<?= htmlspecialchars($value) ?>">
<?php endif; endforeach; ?>
```

## SEO-шаблоны инфоблока

Настраиваются через `MetaTemplate` — позволяют задать шаблон title/description для всех элементов инфоблока или конкретного раздела.

### Установка через миграцию

```php
// В sprint.migration:
$iblockId = $helper->Iblock()->getIblockId('NEWS', 'news');

// SEO-шаблон для элементов инфоблока
\Bitrix\Iblock\Template\Entity\Iblock::setValues($iblockId, [
    'ELEMENT_META_TITLE'       => '{{NAME}} — Новости сайта',
    'ELEMENT_META_DESCRIPTION' => '{{PREVIEW_TEXT}}',
    'ELEMENT_PAGE_TITLE'       => '{{NAME}}',
    'SECTION_META_TITLE'       => '{{NAME}} — Раздел новостей',
    'SECTION_META_DESCRIPTION' => 'Новости раздела {{NAME}}',
]);
```

### Применение в компоненте

Стандартные компоненты `bitrix:news.detail` и `bitrix:catalog.element` применяют SEO-шаблоны автоматически при включённом параметре `SET_TITLE=Y` и `SET_META_TAGS=Y`.

В кастомном компоненте:

```php
use Bitrix\Iblock\Template\Engine;
use Bitrix\Iblock\Template\Entity\Element;

\Bitrix\Main\Loader::includeModule('iblock');

$templateEntity = new Element($elementId);
$title = Engine::process($templateEntity, '{{NAME}} — Купить в интернет-магазине');

global $APPLICATION;
$APPLICATION->SetTitle($title);
```

## Метатеги из свойств элемента инфоблока

```php
// Установить мета из полей элемента вручную
$element = \Bitrix\Iblock\ElementTable::getById($elementId)->fetchObject();

global $APPLICATION;
$APPLICATION->SetTitle($element->getName());
$APPLICATION->SetPageProperty('description', strip_tags($element->getPreviewText()));

// OG-изображение из свойства элемента
$picId = $element->get('PREVIEW_PICTURE');
if ($picId) {
    $src  = 'https://' . $_SERVER['HTTP_HOST'] . \CFile::GetPath($picId);
    $APPLICATION->SetPageProperty('og:image', $src);
}
```

## Robots.txt и Sitemap

Bitrix управляет `robots.txt` через модуль `seo`:

```php
\Bitrix\Main\Loader::includeModule('seo');

// Получить текущий robots.txt сайта
$robotsManager = \Bitrix\Seo\SitemapRobotsFile::getInstance();
$content = $robotsManager->getContent(SITE_ID);
```

Sitemap генерируется через настройки модуля SEO в админке (`/bitrix/admin/seo_sitemap_list.php`). Программно:

```php
\Bitrix\Main\Loader::includeModule('seo');

$sitemap = new \Bitrix\Seo\Sitemap\File(SITE_ID);
$sitemap->generate(); // обновить sitemap
```

## Страницы с запретом индексации

Для страниц поиска, фильтрации, личного кабинета:

```php
global $APPLICATION;
$APPLICATION->SetPageProperty('robots', 'noindex, follow');
```

Или через `noindex`-тег в шаблоне компонента:

```php
// В template.php
?><noindex><?php
// динамический контент, не для индексации
?></noindex><?php
```

## Хлебные крошки

```php
global $APPLICATION;

// Добавить элемент цепочки
$APPLICATION->AddChainItem('Главная', '/');
$APPLICATION->AddChainItem('Каталог', '/catalog/');
$APPLICATION->AddChainItem($element->getName());  // без URL — текущая страница

// Вывести в шаблоне
$APPLICATION->ShowNavChain();  // стандартный вывод
// или
$chain = $APPLICATION->GetNavChain();
foreach ($chain as $item) {
    echo $item['TITLE'], $item['LINK'];
}
```

## Антипаттерны

- Не устанавливать title/description напрямую через `<title>` в шаблоне компонента — только через `$APPLICATION->SetTitle()`.
- Не дублировать мета-теги вручную там, где работают SEO-шаблоны инфоблока.
- Не использовать одинаковый `description` на всех страницах — каждая страница должна иметь уникальное описание.
- Не выводить `og:image` с относительным URL — только абсолютный с протоколом и доменом.
- Не забывать `canonical` на страницах с параметрами (фильтры, пагинация, UTM) — дубли убивают SEO.
