---
title: Выполнение Left Join запросов в LINQ
date: 2021-11-18 10:33:24
tags: [C#, LINQ, LEFTJOIN]
---

Всем хорош подход FluentApi при выполнении запросов в LINQ, но одна вещь ему не по зубам. Это выполнение удобных Left Join запросов. Но SQL-style запросы в LINQ позволяют решить данную проблему.

Конструкция join в LINQ даёт INNER JOIN соединение таблиц (если проводить аналогию с SQL). А для LEFT JOIN рассмотрим код, показывающий как решить данную проблему в LINQ:

```csharp
var salesProducts = 
(
    from product in context.Products
    join person in context.SalesPeople on
    (product.Region, product.Type)
    equals
    (person.Region, person.ProductType)
    into productPeople
    from productPerson in productPeople.DefaultIfEmpty()
    select new 
    {
        Person = productPerson?.Name ?? string.Empty,
        Product = product.Name,
        product.Region,
        product.Type
    }
).ToList();
```

**DefaultIfEmpty()** как раз и осуществляет LEFT JOIN в случае, если *productPerson* не имеет соответствующей сущности в таблице продаванов *SalesPeople*. Результат join запроса, соединяющего сущности таблицы *Products* с сущностями таблицы *SalesPeople* помещается во временную переменную *productPersons* с помощью синтаксической конструкции ```into productPersons``` и затем оператор *DefaultIfEmpty()* в конструкции ```from productPerson in productPersons.DefaultIfEmpty()``` помещает null в качестве *productPerson*, если для указаного продукта не имеется соответствующего продавца. Затем мы формируем с помощью проекции *select* экземпляр анонимного класса, содержащего допустимые значения в случае отсутствия продавца *person* у требуемого продукта *product*. Это достигается с помощью конструкции ```Person = productPerson?.Name ?? string.Empty```, которая позволяет избавиться от последующей проверки значения на null.