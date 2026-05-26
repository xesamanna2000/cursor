---
name: bitrix-psr-12
description: Покрывает кодстайл PHP в Bitrix-проектах по PSR-12 — declare(strict_types=1), именование (PascalCase/camelCase/UPPER_SNAKE_CASE), типизация, final/readonly, use-импорты, форматирование классов и методов, шаблоны (<?=, alternative syntax), комментарии на русском. Применяется при написании и ревью PHP-кода в /local/, создании модулей, сервисов, контроллеров, ORM и компонентов. Ключевые термины — PSR-12, strict_types, camelCase, PascalCase, final, readonly, типизация, шаблоны Bitrix.
---

# PSR-12 в Bitrix-проектах

## Базовые требования

- PHP **8.2+**.
- В каждом PHP-файле с логикой — `declare(strict_types=1);` сразу после открывающего тега.
- Следуй **[PSR-12](https://www.php-fig.org/psr/psr-12/)**; отступы — **табы** (как в проекте).
- Комментарии к неочевидной бизнес-логике — **на русском**.
- Не добавляй очевидных комментариев-нарратива.

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

## Именование

| Сущность | Стиль | Пример |
| :--- | :--- | :--- |
| Папки и классы | PascalCase | `Application/Service/PostService.php` → `PostService` |
| Методы и свойства | camelCase | `getUserId()`, `$createdAt` |
| Поля ORM (`getMap`) | UPPER_SNAKE_CASE | `'USER_ID'`, `'DATE_CREATE'` |
| Константы класса | UPPER_SNAKE_CASE | `public const MAX_ITEMS = 100;` |
| Идентификатор модуля | lower.dot | `vendor.module` |
| Неймспейс модуля | PascalCase | `\Vendor\Module\...` |

## Форматирование по PSR-12

### Открывающий тег и declare

```php
<?php declare(strict_types=1);
```

Полный тег `<?php` — обязателен в файлах логики. Исключение: короткий echo-тег `<?=` только для вывода в шаблонах.

### Фигурные скобки

- Класс, метод, `if`, `foreach`, `try` — открывающая `{` на **новой строке**.
- Закрывающая `}` — на отдельной строке на том же уровне отступа.

```php
if (!\Bitrix\Main\Loader::includeModule('vendor.module'))
{
	throw new \Bitrix\Main\SystemException('Module vendor.module is not installed');
}
```

### use-импорты

- Один `use` на строку, в алфавитном порядке.
- Группы: классы → функции → константы (если есть).
- Не импортируй неиспользуемые символы.

```php
use Bitrix\Main\Loader;
use Bitrix\Main\Result;
use Vendor\Module\Model\PostTable;
```

### Типизация

- Параметры и возвращаемые значения — всегда типизированы.
- Используй `?Type`, `void`, `never`, union-типы и именованные аргументы где уместно.
- Предпочитай `final` для классов, которые не задуманы для наследования.
- Для неизменяемых зависимостей — `readonly` (свойства или promoted parameters).

```php
public function findById(int $id): ?Post
{
	// ...
}

public function rebuild(): void
{
	// ...
}
```

### Конструктор и promoted properties

```php
final class PostService
{
	public function __construct(
		private readonly PostRepository $repository,
		private readonly \Psr\Log\LoggerInterface $logger,
	) {}
}
```

## Шаблоны компонентов и сайта

В `.php`-шаблонах (не в `/lib/`):

- Логика и контрольные структуры — полный тег `<?php`.
- Вывод переменных — короткий echo: `<?=$title?>`, не `<?php echo $title?>`.
- Условия и циклы — alternative syntax:

```php
<?php if ($arResult['ITEMS']): ?>
	<ul>
		<?php foreach ($arResult['ITEMS'] as $item): ?>
			<li><?=htmlspecialcharsbx($item['NAME'])?></li>
		<?php endforeach; ?>
	</ul>
<?php endif; ?>
```

### HTML в шаблонах

- Вложенные теги и содержимое — на отдельных строках с отступами.
- Атрибуты переноси на новые строки, если строка длиннее 120 символов или атрибутов много.

## Специфика Bitrix

### class.php компонента

```php
<?php
if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true)
{
	die();
}

use Bitrix\Main\Localization\Loc;

final class VendorPostListComponent extends \CBitrixComponent
{
	public function executeComponent(): void
	{
		$this->arResult['ITEMS'] = $this->getItems();
		$this->includeComponentTemplate();
	}

	protected function getItems(): array
	{
		return [];
	}
}
```

### ORM DataManager

```php
<?php declare(strict_types=1);

namespace Vendor\Module\Model;

use Bitrix\Main\ORM\Data\DataManager;
use Bitrix\Main\ORM\Fields\IntegerField;
use Bitrix\Main\ORM\Fields\StringField;

final class PostTable extends DataManager
{
	public static function getTableName(): string
	{
		return 'vendor_module_post';
	}

	public static function getMap(): array
	{
		return [
			new IntegerField('ID', [
				'primary' => true,
				'autocomplete' => true,
			]),
			new StringField('TITLE', [
				'required' => true,
			]),
		];
	}
}
```

### install/index.php

Файлы установки модуля могут не иметь `strict_types` (legacy-конвенция Bitrix), но форматирование PSR-12 сохраняй: фигурные скобки на новой строке, табы, типизация публичных методов где возможно.

## Чек-лист перед коммитом

1. `declare(strict_types=1);` в новых файлах `/lib/` и сервисах.
2. Имена классов/методов/полей ORM соответствуют таблице именования.
3. Нет неиспользуемых `use`-импортов (если они мешают задаче — не трогай чужой код).
4. Публичные методы имеют return type; параметры типизированы.
5. В шаблонах — `<?=` для вывода, alternative syntax для условий/циклов.
6. Комментарии только там, где логика неочевидна; текст комментариев — на русском.
