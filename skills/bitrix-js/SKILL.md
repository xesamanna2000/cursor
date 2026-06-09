---
name: bitrix-js
description: Покрывает встроенную JavaScript-библиотеку Bitrix — глобальный объект BX, работа с DOM через BX.find/BX.findChild/BX.create, события BX.bind/BX.ready, пространства имён BX.namespace, AJAX через BX.ajax.runAction и BX.ajax.runComponentAction, утилиты BX.message/BX.clone/BX.type. Применяется при написании клиентского JS в шаблонах компонентов и admin-страницах без использования jQuery. Ключевые термины — BX, BX.ready, BX.find, BX.bind, BX.ajax.runAction, BX.namespace, BX.create.
---

# Встроенная JS-библиотека Bitrix (BX)

`BX` — глобальный объект Bitrix, доступен на всех страницах. Заменяет jQuery и предоставляет DOM-утилиты, AJAX, события и пространства имён.

## Инициализация: BX.ready

```js
BX.ready(function() {
    // DOM готов, можно работать с элементами
    var btn = BX('my-button');
    BX.bind(btn, 'click', function() {
        BX.addClass(btn, 'active');
    });
});
```

## Поиск элементов

```js
// По ID (аналог document.getElementById)
var el = BX('element-id');

// CSS-селектор, первый элемент
var el = BX.find('.my-class');
var el = BX.find(document, '.container .item');  // в рамках контейнера

// Все элементы по CSS-селектору (BX.findAll — main 20+; иначе querySelectorAll)
var items = document.querySelectorAll('.list-item');
var items = container.querySelectorAll('.list-item');

// Поиск дочернего элемента по условию
var child = BX.findChild(parent, { class: 'active' }, true);  // рекурсивно
var child = BX.findChild(parent, { tag: 'INPUT', attr: { name: 'email' } });

// Ближайший родитель по условию
var row = BX.findParent(el, { class: 'b-table-row' });
```

## Создание элементов: BX.create

```js
var div = BX.create('div', {
    attrs: { id: 'my-block', 'data-type': 'card' },
    props: { className: 'card card--active' },
    style: { display: 'flex', marginTop: '10px' },
    text: 'Текст содержимого',   // или html: '<b>HTML</b>'
    events: {
        click: function(e) { BX.PreventDefault(e); }
    },
    children: [
        BX.create('span', { text: 'Дочерний элемент' }),
    ],
});

BX('container').appendChild(div);
```

## DOM: классы, стили, атрибуты

```js
// Классы
BX.addClass(el, 'active');
BX.removeClass(el, 'active');
BX.toggleClass(el, 'active');
BX.hasClass(el, 'active');   // → true/false

// Стили
BX.style(el, 'display', 'none');
BX.style(el, { display: 'flex', color: 'red' });
var display = BX.style(el, 'display');  // получить значение

// Атрибуты
el.setAttribute('data-id', 42);
var val = el.getAttribute('data-id');

// Показать/скрыть
BX.show(el);
BX.hide(el);
BX.toggle(el);
```

## События: BX.bind / BX.unbind / BX.addCustomEvent

```js
// Подписка на DOM-событие
BX.bind(el, 'click', handler);
BX.unbind(el, 'click', handler);

// Делегирование: BX.delegate — только привязка контекста (func, thisObj)
// Для делегирования событий — BX.bindDelegate:
BX.bindDelegate(container, 'click', { className: 'list-item' }, function(e) {
    var item = BX.findParent(e.target, { class: 'list-item' });
    // ...
});

// Предотвращение всплытия и дефолтного поведения
BX.PreventDefault(e);
BX.stopPropagation(e);

// Кастомные события (pub/sub)
BX.addCustomEvent('MyModule:itemSelected', function(data) {
    console.log(data.id);
});

BX.onCustomEvent('MyModule:itemSelected', { id: 5 });
```

## Пространства имён: BX.namespace

```js
// Объявить namespace (создаёт вложенные объекты, если не существуют)
BX.namespace('MyVendor.MyModule');

MyVendor.MyModule.Widget = function(options) {
    this.container = BX(options.containerId);
    this.init();
};

MyVendor.MyModule.Widget.prototype = {
    init: function() {
        BX.bind(this.container, 'click', BX.delegate(this.onClick, this));
    },
    onClick: function(e) {
        BX.PreventDefault(e);
        // ...
    },
};

// Инициализация из PHP-шаблона
// new MyVendor.MyModule.Widget({ containerId: 'widget-<?= $this->randString() ?>' });
```

## AJAX: BX.ajax.runAction

Вызов action-метода D7-контроллера:

```js
BX.ajax.runAction('vendor.module.post.create', {
    data: {
        title: BX('input-title').value,
        body:  BX('textarea-body').value,
    },
})
.then(function(response) {
    console.log(response.data);  // возвращаемое значение action-метода
})
.catch(function(response) {
    console.error(response.errors);
});
```

Формат action: `<vendor>.<module>.<controller>.<action>` — нижний регистр, без суффикса `Action`. CSRF-токен добавляется автоматически.

## AJAX: BX.ajax.runComponentAction

Вызов action-метода Controllerable-компонента:

```js
BX.ajax.runComponentAction('vendor:my.component', 'getItems', {
    mode: 'class',
    data: { page: 2 },
})
.then(function(response) {
    // response.data
});
```

## Локализация: BX.message

Передача переводов из PHP в JS:

```php
// В template.php компонента (после Loc::loadMessages)
\Bitrix\Main\Localization\Loc::loadMessages(__FILE__);
?>
<script>
BX.message({
    MY_PHRASE: '<?= \CUtil::JSEscape(\Bitrix\Main\Localization\Loc::getMessage('MY_PHRASE')) ?>',
    CONFIRM_DELETE: '<?= \CUtil::JSEscape(\Bitrix\Main\Localization\Loc::getMessage('CONFIRM_DELETE')) ?>',
});
</script>
```

В JS:

```js
alert(BX.message('MY_PHRASE'));
```

## Утилиты

```js
// Тип значения
BX.type.isString(val);
BX.type.isArray(val);
BX.type.isFunction(val);
BX.type.isInteger(val);
BX.type.isDomNode(val);

// Расширение объекта (shallow merge)
var merged = BX.mergeEx(defaults, options);

// Глубокое клонирование
var copy = BX.clone(obj, true);

// Экранирование для HTML
var safe = BX.util.htmlspecialchars(userInput);

// Случайная строка (для уникальных ID)
var uid = BX.util.getRandomString(8);

// Сериализация формы в объект
var data = BX.ajax.serialize(BX('my-form'));
```

## Подключение JS в шаблоне

Подробнее про `Asset`, `Extension::load`, порядок загрузки — скилл `bitrix-assets`.

```php
\Bitrix\Main\Page\Asset::getInstance()->addJs('/local/js/vendor/my-script.js');
\Bitrix\Main\UI\Extension::load('ui.notification');
```

## Антипаттерны

- Не использовать jQuery — в современном Bitrix он не гарантирован; заменять на `BX`-аналоги.
- Не писать глобальные переменные без `BX.namespace` — конфликты неизбежны при нескольких компонентах.
- Не вставлять пользовательский ввод напрямую в HTML (`innerHTML`) — только через `BX.create` или `BX.util.htmlspecialchars`.
- Не передавать `sessid` вручную в `BX.ajax.runAction` — он добавляется автоматически.
- Не инициализировать виджеты до `BX.ready` — элементы могут ещё не существовать в DOM.
