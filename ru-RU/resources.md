# Ресурсы (модульность)

Основой модульности `basis.js` являются ресурсы.

Ресурс - это некоторый файл. Ресурсы используются для того, чтобы выносить из javascript контент разного типа в отдельные файлы, а также сегментировать сам javascript код.

Для определения ресурса используется функция `basis.resource()`, которой передается путь к файлу. Если путь относительный (не начинается с `/`), то он разрешается относительно корневого `html` файла (обычно это `index.html`).

Результатом вызова `basis.resource` будет специальная функция, которая возвращает содержимое файла. Такая функция перенимает интерфейс у [`basis.Token`](basis.Token.md) и поддерживает механизм [`binding bridge`](bindingbridge.md). Первый её вызов приводит к загрузке файла и его кешированию, а последующие вызовы лишь возвращают закешированный результат. У такой функции также есть метод `fetch`, который делает то же, что и вызов самой функции, и используется для улучшения читаемости кода.

```js
var someText = basis.resource('./path/to/file.txt'); // объявление, файл еще не загружен

console.log(someText());        // файл будет загружен, его содержимое будет закешировано и возвращено
console.log(someText.fetch());  // эквивалент, будет возвращено закешированное значение
```

Ресурсы обеспечивают возможность раннего связывания и поздней инициализации. Пример:

```js
var MyControl = basis.ui.Node.subclass({
  template: basis.resource('./path/to/template.tmpl')
});
```

Здесь создается класс `MyControl` и определяется файл шаблона. Такое определение не приводит к загрузке файла шаблона, так как он еще не нужен. Но когда будет создан первый экземпляр этого класса, потребуется создать экземпляр шаблона, и только тогда будет загружен файл и использовано его содержимое.

В зависимости от типа файла (его расширения) может возвращаться не только текстовое значение файла, но и значения других типов. То, что будет возвращаться, определяет post-обработчик, ассоциированный с определенным расширением. В `basis` определены обработчики для расширений `.js`, `.json` и `.css`. Можно определить и собственные обработчики.

## JavaScript

Содержимое `.js` файлов оборачивается в специальную функцию и немедленно вызывается с определенными параметрами. Таким образом, для кода всегда создается локальная область видимости, в которой доступен ряд значений и функций.

В область видимости кода добавляются следующие значения:

* `exports` – объект экспорта;
* `module` – объект, представляющий модуль;
* `basis` – ссылка на корневое пространство имен basis.js;
* `global` – ссылка на глобальную область видимости (в браузере это `window`);
* `__filename` – полный путь и имя файла;
* `__dirname` – полный путь до директории, содержащей файл;
* `resource` – функция, которая делает то же, что и `basis.resource`, но разрешает пути к файлам относительно текущего файла, а не корневого `html` файла.

Пример модуля, допустим, его путь `/src/module/list/index.js`:

```js
basis.require('basis.ui');

var list = basis.ui.Node({
  template: resource('./template/list.tmpl'),  // путь к файлу /src/module/list/template/list.tmpl
  ...
});

module.exports = {      // то, что будет возвращаться при использовании ресурса
  MyClass: MyClass
};
```

Использование:

```js
// объявляем ресурс, файл еще не загружен и код не выполнен
var list = basis.resource('./src/module/list/index.js');
console.log(list());    // файл загружается, выполняется код, возвращается module.exports

// либо
var list = basis.resource('./src/module/list/index.js').fetch();  // файл загружается,
console.log(list);      // { MyClass: ..., someFn: ... }
```

Следующий код схематично показывает, что происходит с содержимым javascript-файла:

```js
var module = {
  exports: {}
};
var relResource = function(url){
  return basis.resource('./src/module/list/' + url);
};

(function(exports, module, basis, global, __filename, __dirname, resource){
  'use strict';

  // содержимое файла

}).call(module.exports, module.exports, module, basis, this,
  'src/module/list/index.js', 'src/module/list/', relResource);

return module.exports;
```

Похожим образом работает функция `require` в `node.js`.

> Код `javascript`-ресурсов выполняется в `strict mode`.

## JSON

Если файл имеет расширение `.json`, то содержимое файла парсится функцией `JSON.parse` и возвращается результат.

```js
var settings = basis.resource('./settings.json').fetch();
if (settings.someName) {
  // do something
}
```

Если файл содержит некорректный `json`, то будет возвращен `null`.

## CSS

Для файлов с расширением `.css` создается и возвращается специальная обертка, экземпляр класса `CssResource`.

Такая обертка имеет два основных метода `startUse` и `stopUse`. При первом вызове `startUse` в документ добавляется элемент `<style>` содержащий код `.css` файла. При этом считается количество вызовов метода `startUse`, и если столько же раз будет вызван метод `stopUse`, то элемент `<style>` будет удален из документа. При изменении содержимого `.css` файла, содержимое тега `<style>` так же обновляется.

Вставка содержимого в тег `<style>` производится таким образом, чтобы ссылки на ресурсы (`@import`, `url(..)` и т.д.) разрешались относительно папки расположения файла. Так же, начиная с версии `1.2.4`, к содержимому добавляется `//@ sourceURL=..`, чтобы инструменты разработки, такие как `Developer Tools` в `Google Chrome`, могли ассоциировать сождержимое `<style>` с оригинальным файлом.

Получить содержимое файла можно обратившись к свойству `cssText`.

```js
var myStyle = basis.resource('./style.css').fetch();
console.log(myStyle.cssText); // выведет содержимое файла
myStyle.startUse();  // добавит в документ <style>
...
myStyle.stopUse();   // удалит <style> из документа
```

## Добавление собственных обработчиков ресурсов

Чтобы определить собственный обработчик для некоторого расширения, нужно зарегистирировать его в объекте `basis.resource.extensions`, где ключ - это расширение файла, начинающееся с точки, а значение - функция, принимающая содержимое файла и путь к нему. Возвращаемое такой функцией значение будет являться значением ресурса.

Допустим, требуется добавить поддержку `CoffeeScript`. Для этого нужно подключить компилятор `CoffeeScript` и назначить обработчик расширений `.coffee`, например, так:

```html
<script src="path/to/coffeescript.js"></script>
<script>
  basis.resource.extensions['.coffee'] = function(content, url){
    return basis.resource.extensions['.js'](CoffeeScript.compile(content), url);
  }
</script>
```

Компилятор `CoffeeScript` можно подключить через механизм ресурсов:

```js
var CoffeeScript = basis.resource('./path/to/coffeescript.js').fetch().CoffeeScript;

basis.resource.extensions['.coffee'] = function(content, url){
  return basis.resource.extensions['.js'](CoffeeScript.compile(content), url);
}
```

> Не все стороние библиотеки могут быть подключены как ресурс, так как не все они работают в strict mode.

Данное решение будет работать для `dev` режима, но сборщиком `CoffeeScript` будет восприниматься как строка, следовательно, код такого модуля не будет проанализирован, а его зависимости не будут найдены. Для решения проблемы сборщику нужно скомпилировать `CoffeeScript` в `javascript`, для этого в его настройках определяется препроцессор для файлов с расширением `.coffee`:

```json
{
  "build": {
    ...
    "extFileTypes": {
      ".coffee": {
        "type": "script",
        "preprocess": "compile-coffee-script.js"
      }
    }
  }
}
```

Содержимое `compile-coffee-script.js` может иметь такой вид:

```js
var CoffeeScript = require('./path/to/coffeescript.js');

exports.process = function(content, file, baseURI, console){
  console.log('Compile ' + file.relpath + ' to javascript');
  file.filename = file.filename.replace(/.coffee$/i, '.js');
  return CoffeeScript.compile(content);
}
```

С таким препроцессором в сборке `CoffeeScript` код будет скомпилирован в `javascript`, а расширение файлов изменено с `.coffee` на `.js`.
