---
title: LINQ оператор OrderBy
date: 2022-03-31 12:49:02
tags: [C#, LINQ, OrderBy]
---

Для сортировки последовательности в LINQ имеется четыре метода:

``` csharp
IOrderedEnumerable<T> OrderBy<T>(this IEnumerable<T> items, Func<T, K> keySelector);
IOrderedEnumerable<T> OrderByDescending<T>(this IEnumerable<T> items, Func<T, K> keySelector);

IOrderedEnumerable<T> ThenBy<T>(this IOrderedEnumerable<T> items, Func<T, K> keySelector);
IOrderedEnumerable<T> ThenByDescending<T>(this IOrderedEnumerable<T> items, Func<T, K> keySelector);
```

Первые два дают на выходе последовательность, упорядоченную по возрастанию/убыванию ключей. А keySelector — это как раз функция, которая каждому элементу последовательности ставит в соответствие некоторый ключ, по которому его будут сравнивать при сортировке.

``` csharp
var names = new[] { "Pavel", "Alexander", "Anna" };
IOrderedEnumerable<string> sorted = names.OrderBy(n => n.Length);
Assert.That(sorted, Is.EqualTo(new[] { "Anna", "Pavel", "Alexander" }));
```

Если при равенстве ключей вы хотите отсортировать элементы по другому критерию, то на помощь приходит метод ThenBy.

Например, в следующем примере все имена сортируются по убыванию длин, а при равных длинах — лексикографически.

``` csharp
var names = new[] { "Pavel", "Alexander", "Irina" };

var sorted = names
    .OrderByDescending(name => name.Length)
    .ThenBy(n => n);

Assert.That(sorted, Is.EqualTo(new[] { "Alexander", "Irina", "Pavel" }).AsCollection);
```

Чтобы убрать из последовательности все повторяющиеся элементы используют LINQ функцию Distinct:

``` csharp
var numbers = new[] { 1, 2, 3, 3, 1, 1, };
var uniqueNumbers = numbers.Distinct();
Assert.That(uniqueNumbers, Is.EqualTo(new[] { 1, 2, 3 }).AsCollection));
```
