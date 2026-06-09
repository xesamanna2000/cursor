---
name: bitrix-file
description: Покрывает работу с файлами в Bitrix — CFile для хранения файлов в БД и на диске, \Bitrix\Main\IO\File и IO\Directory для файловой системы, загрузка через форму, получение URL, удаление, привязка к ORM-полю типа FileField. Применяется при загрузке изображений и документов, хранении файлов в сущностях, работе с файловой системой. Ключевые термины — CFile, FileField, IO\File, SaveFile, GetPath, DeleteFile.
---

# Работа с файлами в Bitrix

Два уровня работы с файлами:

| Уровень | Класс | Когда использовать |
|---|---|---|
| Файлы в БД | `CFile` | Загрузка через форму, хранение в ORM-сущности, получение URL |
| Файловая система | `\Bitrix\Main\IO\File` / `IO\Directory` | Чтение/запись файлов напрямую без привязки к БД |

## CFile — файлы в базе данных

Bitrix хранит метаданные файла в таблице `b_file` и физически кладёт файл в `/upload/`. `CFile` — API для этой системы.

### Загрузка файла из формы

```php
$request = \Bitrix\Main\Application::getInstance()->getContext()->getRequest();
$fileData = $request->getFile('PICTURE');  // никогда не используй $_FILES напрямую

if (!empty($fileData['name'])) {
    $fileId = \CFile::SaveFile(
        \CFile::MakeFileArray($fileData['tmp_name'], $fileData['type'], $fileData['name']),
        'vendor/module/pictures'   // подпапка в /upload/
    );

    if (!$fileId) {
        // ошибка сохранения
    }
}
```

### Получение URL файла

```php
// Путь относительно корня сайта
$src = \CFile::GetPath($fileId);        // /upload/vendor/module/pictures/image.jpg

// Полный URL
$serverName = \Bitrix\Main\Application::getInstance()->getContext()->getServer()->getServerName();
$url = 'https://' . $serverName . \CFile::GetPath($fileId);

// Массив с полными данными файла
$fileInfo = \CFile::GetFileArray($fileId);
// ['ID', 'FILE_NAME', 'CONTENT_TYPE', 'FILE_SIZE', 'HEIGHT', 'WIDTH', 'SRC' => '/upload/...']
```

### Удаление файла

```php
\CFile::Delete($fileId);
```

При обновлении сущности — сначала удалить старый файл, потом сохранить новый:

```php
if ($oldFileId && $oldFileId !== $newFileId) {
    \CFile::Delete($oldFileId);
}
```

### Привязка к ORM-сущности (FileField)

```php
use Bitrix\Main\ORM\Fields\FileField;

// В getMap() таблета:
(new FileField('PICTURE'))
    ->configureTitle('Изображение')
    ->configureNullable(),
```

ORM сам сохраняет файл при `add()`/`update()` и возвращает ID в поле. Передавать нужно массив файла:

```php
PostTable::add([
    'TITLE'   => 'Заголовок',
    'PICTURE' => \CFile::MakeFileArray('/path/to/image.jpg'),
]);

// Или из загруженного файла формы:
$fileData = \Bitrix\Main\Application::getInstance()->getContext()->getRequest()->getFile('PICTURE');
PostTable::update($id, [
    'PICTURE' => $fileData,  // ORM сам обработает
]);
```

### Изменение размера изображения

```php
// Создать уменьшенную копию
$resized = \CFile::ResizeImageGet(
    $fileId,
    ['width' => 800, 'height' => 600],
    BX_RESIZE_IMAGE_PROPORTIONAL,  // или BX_RESIZE_IMAGE_EXACT
    true,                           // создавать файл на диске
);
// $resized['src'] — URL уменьшенного изображения
```

## IO\File — прямая работа с файловой системой

Для чтения/записи файлов без привязки к БД.

```php
use Bitrix\Main\IO\File;
use Bitrix\Main\IO\Directory;

// Проверка существования
if (File::isFileExists('/var/log/bitrix/import.log')) {
    // ...
}

// Чтение
$file = new File($_SERVER['DOCUMENT_ROOT'] . '/local/data/config.json');
$content = $file->getContents();
$data = json_decode($content, true);

// Запись (перезаписывает)
$file->putContents(json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE));

// Добавление в конец файла
$file->putContents($newLine . PHP_EOL, File::APPEND);

// Удаление файла
$file->delete();

// Получить имя и расширение
$file->getName();       // config.json
$file->getExtension();  // json
$file->getSize();       // в байтах
```

## IO\Directory — работа с папками

```php
use Bitrix\Main\IO\Directory;

$dir = new Directory($_SERVER['DOCUMENT_ROOT'] . '/local/data/exports/');

// Создать папку (рекурсивно)
if (!$dir->isExists()) {
    Directory::createDirectory($dir->getPath());
}

// Перебрать содержимое
foreach ($dir->getChildren() as $child) {
    if ($child instanceof File) {
        echo $child->getName();
    }
}

// Удалить папку со всем содержимым
$dir->delete();
```

## Валидация загружаемых файлов

```php
$allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
$maxSize = 5 * 1024 * 1024; // 5 МБ

$fileData = $request->getFile('IMAGE');

if (empty($fileData['name'])) {
    return (new \Bitrix\Main\Result())->addError(new \Bitrix\Main\Error('Файл не загружен'));
}

if (!in_array($fileData['type'], $allowedTypes, true)) {
    return (new \Bitrix\Main\Result())->addError(new \Bitrix\Main\Error('Недопустимый тип файла'));
}

if ($fileData['size'] > $maxSize) {
    return (new \Bitrix\Main\Result())->addError(new \Bitrix\Main\Error('Файл слишком большой'));
}

if (!\CFile::CheckImageFile($fileData)) {
    return (new \Bitrix\Main\Result())->addError(new \Bitrix\Main\Error('Файл не является изображением'));
}
```

## Антипаттерны

- Не обращаться к `$_FILES` напрямую — использовать `$request->getFile()`.
- Не хранить файлы вне `/upload/` при использовании `CFile` — пути генерируются автоматически.
- Не забывать удалять старый файл из `b_file` при обновлении — иначе накапливается мусор на диске.
- Не использовать `file_get_contents`/`file_put_contents` напрямую — `IO\File` правильно обрабатывает права и пути.
- Не передавать пользовательское имя файла в путь без санитизации — только через `CFile::SaveFile`.
