# Наборы

Набор (dataset) – *неупорядоченное множество* объектов данных (на данный момент экземпляров `basis.data.Object` и его потомков). Они отвечают на вопрос: входит ли некоторый объект в их состав или нет. Так как набор это множество, то каждый объект в наборе присутствует в единственном экземпляре.

Основополагающим классом этой группы является `basis.data.AbstractDataset`.

Наборы бывают двух типов: статичные (пользовательские), которые имеют интерфейс для изменения состава множества, и автоматические наборы, состав которых определяется некоторым правилом.

Основные классы наборов описаны в пространстве имен `basis.data`, дополнительные в `basis.data.dataset` (большая часть автоматических наборов), `basis.data.index` и других.

## basis.data.AbstractDataset

`AbstractDataset` является освопологающим классом для наборов, наследуется от `basis.data.AbstactData`. Экземпляры класса может хранить данные, но не имеют интерфейса для модификации состава.

> До версии 1.0.0 `AbstractDataset` наследовался от `basis.data.Object`.

Наборы хранят элементы в виде карты или более сложных структур, поэтому чтобы получить список всех элементов набора, используется метод `getItems`. Количество элементов можно узнать с помощью свойства `itemCount`. Чтобы узнать входит ли элемент в состав набора, используется метод `has`.

Чтобы получить любой произвольный элемент набора используется метод `pick`, чтобы получить N (максимум) произвольных элементов метод `top`.

```js
console.log(dataset.getItems());
// console> [object, object, object]

console.log(dataset.itemCount);
// console> 3

console.log(dataset.pick());
// console> object

console.log(dataset.top(2));
// console> [object, object]

console.log(dataset.has(dataset.pick()));
// console> true
```

Метод `forEach` позволяет выполнить некоторую функцию для все элементых набора.

```js
dataset.forEach(function(item){
  console.log(item);
});
```

Когда меняется состав набора, выбрасывается событие `itemsChanged`. Обработчики получают параметр `delta` - изменения в наборе. Если добавлены новые элементы, то дельта содержит свойство `inserted`, массив с новыми членами, а если некоторые элементы удалены, то дельта содержит свойство `deleted`, массив с удаленными членами.

```js
someDataset.addHandler({
  itemsChanged: function(sender, delta){
    if (delta.inserted)
      for (var i = 0, item; item = delta.inserted[i]; i++)
        console.log('item added', delta.inserted[i]);

    if (delta.deleted)
      for (var i = 0, item; item = delta.deleted[i]; i++)
        console.log('item removed', delta.deleted[i]);
  }
});
```

## Dataset

Класс `Dataset` предназначен для создания пользовательских наборов. Этот класс имеет методы для прямого управления составом набора.

Основые методы с набором (методы):

  * `add(items)` - добавление элементов; 

  * `remove(items)` - удаление элементов;

  * `set(items)` - задать набор;

  * `sync(items)` - синхронизация с набором (см. подробнее ниже);

  * `clear()` - очистить набор;

Методам `add`, `remove`, `set`, `clear` передается массив элементов. Для методов `add` и `remove` можно передать и одиночный объект. При выполнении методов дубликаты и значения не являющиеся экземплярами `basis.Data.DataObject` – игнорируются.

В ходе выполнения методов составляется дельта изменений: какие элементы удалены и какие добавлены. Если дельта не пустая (хотя бы один элемент добавился или удалился), то выбрасывается событие `itemsChanged`.

Процесс синхронизации (метод `sync`) подразумевает сравнение текущего набора с переданным. В случае удаления элемента из набора, он разрушается (вызывается его метод `destroy`).

> Пусть текущий набор A, а переданный набор B. Если элемент присутствует в обоих наборах, то никаких изменений. Если элемент есть только в A, но отсутствует в B, то элемент разрушается. Если элемента нет в A, но есть в B, то он добавляется в A.

Можно задать состав набора при создании, для этого используется свойство `items`.

```js
var dataset = new basis.data.Dataset({
  items: [
    ...
  ]
});
```
