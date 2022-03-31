---
title: LINQ оператор SelectMany
date: 2022-03-22 09:14:08
tags: [C#, LINQ, SelectMany]
---

В LINQ есть такой необычный оператор, как ```SelectMany```. В чем же его необычность?

Рассмотрим сигнатуру метода:

``` csharp
IEnumerable<R> SelectMany(this IEnumerable<T> items, Func<T, IEnumerable<R>> f)
```

Из сигнатуры метода ```SelectMany``` видно, что он применяется к последовательности типа ```IEnumerable<T>```. К каждому элементу этой последовательности применяется функция ```Func<T, IEnumerable<R>>```, которая преобразует элемент типа ```T``` в последовательность типа ```IEnumerable<R>```. И на выходе мы имеем последовательность типа ```IEnumerable<R>```. Получается, что каждый элемент превращается в множество, которое может быть закрыто отличным типом от исходного, которое затем спрямляется. Результатом работы ```SelectMany``` является конкатенация всех полученных последовательностей, т.е. мы имеем не последовательность последовательностей, а единую последовательность типа, отличного от исходного (не обязательно отличного, можно получить и последовательность того же типа).

Поясним вышесказанное следующим примером:

``` csharp
string[] words = {"ab", "", "c", "de"};
IEnumerable<char> letters = words.SelectMany(w => w.ToCharArray());
Assert.That(letters, Is.EqualTo(new[] {'a', 'b', 'c', 'd', 'e'}));
```

Впрочем строка уже сама по себе является последовательностью символов и реализует интерфейс IEnumerable<char>, поэтому вызов ToCharArray на самом деле лишний.

Одно из не совсем очевидных применений ```SelectMany``` — это вычисление декартова произведения двух множеств. Декартово произведение множества {-1, 0, 1} на само себя даст все возможные относительные координаты соседей условной точки Point, где Point -- класс, имеющий координаты X и Y в качестве открытых полей. Для вычисления декартова произведения двух множеств также потребуется использовать метод ```Select``` внутри ```SelectMany```, как показано в примере ниже:

``` csharp
public static IEnumerable<Point> GetNeighbours(Point p)
{
    int[] d = {-1, 0, 1};
    return d.SelectMany(i => d.Select(s => new Point(p.X + i, p.Y + s)));
}
```

Вот ещё один интересный пример использования ```SelectMany```. Требуется составить лексикографически упорядоченный список всех встречающихся слов в массиве строк. Слова нужно сравнивать регистронезависимо и выводить в нижнем регистре.

``` csharp
public static void Main()
{
 var vocabulary = GetSortedWords(
  "Hello, hello, hello, how low",
  "",
  "With the lights out, it's less dangerous",
  "Here we are now; entertain us",
  "I feel stupid and contagious",
  "Here we are now; entertain us",
  "A mulatto, an albino, a mosquito, my libido...",
  "Yeah, hey"
 );
 foreach (var word in vocabulary)
  Console.WriteLine(word);
}

public static string[] GetSortedWords(params string[] textLines)
{
 return textLines
       .SelectMany(x => Regex.Split(x.ToLower(), @"\W+"))
       .Where(x => x.Length > 0)
       .Distinct()
       .OrderBy(x => x)
       .ToArray();
}
```

Здесь для разбиения строки на слова используется класс ```Regex``` из пространства имён ```System.Text.RegularExpressions```. Все полученные слова "спрямляются" (объединяются из разных массивов в один массив), удаляются пустые строки и повторы. В конце массив лексикографически упорядочивается и возвращается.

Без использования ```SelectMany``` пришлось бы очень трудно, т.к. каждая входящая строка после разбиения ```Regex.Split``` превращается в массив слов и мы бы получили массив массивов.

Вывод программы можно увидеть ниже:

```
a
albino
an
and
are
contagious
dangerous
entertain
feel
hello
here
hey
how
i
it
less
libido
lights
low
mosquito
mulatto
my
now
out
s
stupid
the
us
we
with
yeah
```
