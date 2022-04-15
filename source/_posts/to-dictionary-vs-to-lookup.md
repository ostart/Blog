---
title: ToDictionary vs ToLookup
date: 2022-04-15 10:52:53
tags: [C#, LINQ, ToDictionary, ToLookup]
---

Встречается необходимость, сгруппировав элементы, преобразовать их в структуру данных для поиска группы по ключу группировки. Это можно сделать, например, так:

``` csharp
string[] names = {"Pavel", "Peter", "Andrew", "Anna", "Alice", "John"};
var namesByLetter = new Dictionary<char, List<string>>();

foreach (var group in names.GroupBy(name => name[0]))
	namesByLetter.Add(group.Key, group.ToList());

Assert.That(namesByLetter['J'], Is.EquivalentTo(new[] { "John" }));
Assert.That(namesByLetter['P'], Is.EquivalentTo(new[] {"Pavel", "Peter"}));
Assert.IsFalse(namesByLetter.ContainsKey('Z'));
```

Ровно того же эффекта можно добиться и без цикла при помощи LINQ-метода ```ToDictionary```, имеющего следующую сигнатуру:

``` csharp
IDictionary<K, V> ToDictionary(this IEnumerable<T> items, Func<T, K> keySelector, Func<T, V> valueSelector)
```

Тогда предыдущий пример с ```foreach``` в случае ```ToDictionary``` примет следующий вид:

``` csharp
string[] names = {"Pavel", "Peter", "Andrew", "Anna", "Alice", "John"};

Dictionary<char, List<string>> namesByLetter = names
	.GroupBy(name => name[0])
	.ToDictionary(group => group.Key, group => group.ToList());

Assert.That(namesByLetter['J'], Is.EquivalentTo(new[] { "John" }));
Assert.That(namesByLetter['P'], Is.EquivalentTo(new[] {"Pavel", "Peter"}));
Assert.IsFalse(namesByLetter.ContainsKey('Z'));
```

Ещё проще воспользоваться LINQ-методом ```ToLookup```, имеющим следующие сигнатуры:

``` csharp
ILookup<K, T> ToLookup(this IEnumerable<T> items, Func<T, K> keySelector)
```
``` csharp
ILookup<K, V> ToLookup(this IEnumerable<T> items, Func<T, K> keySelector, Func<T, V> valueSelector)
```

В отличие от словаря, ```Lookup``` является неизменяемым (immutable) типом. У него нет методов типа ```Add``` и открытого конструктора. Интересно, что ```Lookup``` по неизвестному ключу возвращает пустую коллекцию, а Dictionary в такой ситуации выбрасывает исключение. Также в ```Lookup``` можно использовать ключи типа ```null```.
Замечу, что операции ```list.ToLookup(x => x)``` и ```list.GroupBy(x => x).ToDictionary(group => group.Key)``` семантически эквивалентны.

Пример с ```foreach``` и ```ToDictionary``` в случае ```ToLookup``` примет следующий вид:

``` csharp
string[] names = {"Pavel", "Peter", "Andrew", "Anna", "Alice", "John"};

ILookup<char, string> namesByLetter = names.ToLookup(name => name[0], name => name.ToLower());

Assert.That(namesByLetter['J'], Is.EquivalentTo(new[] {"john"}));
Assert.That(namesByLetter['P'], Is.EquivalentTo(new[] {"pavel", "peter"}));
			
// Lookup по неизвестному ключу возвращает пустую коллекцию. 
// Это бывает удобнее, чем поведение Dictionary, который в такой ситуации бросает исключение.
Assert.That(namesByLetter['Z'], Is.Empty);
```

В русскоязычной литературе ```Lookup``` раньше именовался таблицей истинности. ```Lookup``` можно использовать в построении обратного индекса. Обратный индекс — это структура данных, часто использующаяся в задачах полнотекстового поиска нужного документа в большой базе документов. По своей сути обратный индекс напоминает индекс в конце бумажных энциклопедий, где для каждого ключевого слова указан список страниц, где оно встречается.

Допустим наш документ определён так:

``` csharp
public class Document
{
	public int Id;
	public string Text;
}
```

Тогда для списка документов обратный индекс можно получить так:

``` csharp
public static ILookup<string, int> BuildInvertedIndex(Document[] documents)
{
    return documents
        .SelectMany(x => Regex.Split(x.Text, @"\W+")
		    .Where(x => x.Length > 0)
		        .Select(y => Tuple.Create(y.ToLower(), x.Id)))
		            .Distinct()
        .ToLookup(x => x.Item1, x => x.Item2);
}
```
