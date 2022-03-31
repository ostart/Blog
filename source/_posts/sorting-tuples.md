---
title: Сортировка кортежей
date: 2022-03-31 13:45:30
tags: [C#]
---

Создание кортежа с помощью конструктора выглядит громоздко. Чтобы облегчить синтаксис создания кортежей существует класс ```Tuple``` с серией статических методов, создающих кортежи:

``` csharp
var t1 = Tuple.Create(42, "abc");
// Эквивалентно:
// var t1 = new Tuple<int, string>(42, "abc");
```

Полезное свойство кортежей состоит в том, что они реализуют интерфейс ```IComparable```, сравнивающий кортежи по компонентам. То есть Tuple.Create(1, 2) будет меньше Tuple.Create(2, 1). Этот интерфейс по умолчанию используется в методах сортировки и поиска минимума/максимума.

Использование данного факта рассмотрим на примере:

Дан текст. Нужно составить список всех встречающихся в тексте слов, упорядоченный сначала по возрастанию длины слова, а потом лексикографически. Удивительно, но данную задачу можно решить совсем не используя ```ThenBy```.

Наивное решение заключается в том, чтобы сформировать нужные кортежи, отсортировать их, а затем извлечь вторые компоненты кортежей, содержащих решение:

``` csharp
public static List<string>  GetSortedWords(string text)
{
    return Regex.Split(text, @"\W+")
                .Where(x => x.Length > 0)
                .Select(t => Tuple.Create(t.Length, t.ToLowerInvariant()))
                .OrderBy(x => x)
                .Select(x => x.Item2)
                .Distinct()
                .ToList();
}
```

Однако следует напомнить, что keySelector функции OrderBy — это функция, которая каждому элементу последовательности ставит в соответствие некоторый ключ, по которому его будут сравнивать при сортировке. Поэтому решение можно записать намного короче, поместив кортеж (типа ValueTuple) в keySelector функции OrderBy:

``` csharp
public static List<string>  GetSortedWords(string text)
{
    return Regex.Split(text.ToLower(), @"\W+")
                .Where(x => !string.IsNullOrEmpty(x))
                .OrderBy(x => (x.Length, x))
                .Distinct()
                .ToList();
}
```

Вывод программы можно увидеть ниже:

```
GetSortedWords("A box of biscuits, a box of mixed biscuits, and a biscuit mixer.")
'a' 'of' 'and' 'box' 'mixed' 'mixer' 'biscuit' 'biscuits'

GetSortedWords("Each Easter Eddie eats eighty Easter eggs.")
'each' 'eats' 'eggs' 'eddie' 'easter' 'eighty'
```