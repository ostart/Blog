---
title: Метод Contains больше не поддерживается в EF Core 6
date: 2023-04-13 16:40:03
tags: [C#, EntityFramework, EFCore5, EFCore6, LINQ, .NET5, .NET6]
---

При переходе с ```.NET 5``` на ```.NET 6``` столкнулся с такой проблемой: ```EF Core``` отказался строить сложные ```IQueryable``` запросы LINQ в ```.NET 6```, с которыми ранее справлялся в ```.NET 5``` без проблем. Поиски ответа на вопрос, что же случилось, привели к следующему решению.

При использовании ```.NET 5``` и ниже использование метода ```Contains``` для проверки содержится ли значение в множестве других значений препокойно транслировалось в ```SQL``` выражение вида ```WHERE City IN ('Paris','London')```, но начиная с версии ```.NET 6``` метод ```Contains``` не может как раньше транслироваться в ```SQL``` и падает с исключением, предлагающим упростить выражение или переписать его через другие операторы.

Таким образом, такой код больше не работает в ```.NET 6```:

``` csharp
var ids = new List<int> { 1, 2, 3 };
var entities = dbContext.Posts.Where(p => ids.Contains(p.Id));
```

Решение заключается в использовании метода ```Any``` вместо ```Contains``` для проверки содержится ли значение в множестве других значений:

``` csharp
var ids = new List<int> { 1, 2, 3 };
var entities = dbContext.Posts.Where(p => ids.Any(id => id == p.Id));
```

Вот такое LINQ выражение ```.NET 6``` уже принимает спокойно и ```EF Core``` транслирует в ту же самую конструкцию ```SQL``` вида ```WHERE Id IN (1,2,3)```.

Переписав все конструкции содержащие ```ids.Contains``` в ```ids.Any``` всё стало работать как и прежде, и задача была успешно решена.

Хотел бы отметить, что данная проблема касается только выборки из множества значений. ```String.Contains``` данная проблема не коснулась и он нормально транслируется в конструкцию ```SQL```  вида ```LIKE```.