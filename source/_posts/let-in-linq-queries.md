---
title: Использование let для упрощения LINQ запросов
date: 2021-12-14 16:47:36
tags: [C#, LINQ]
---

Бывает при выполнении LINQ запросов требуются промежуточные преобразования с переменными запроса. Обычно это достигается с помощью LINQ оператора let. 

Мощь этого оператора рассмотрим на следующем примере. Допустим у нас есть класс Employee вида:
``` csharp
public class Employee
{
    public int ID { get; set; }
    public string Name { get; set; }
    public string Department { get; set; }
    public string Address { get; set; }
    public string PostalCode { get; set; }
    public string Salary { get; set; }
}
```

Тогда запрос с промежуточными преобразованиями может выглядить так:
``` csharp
var employeesWithAddress = 
    (
        from employee in Employees
        let fullAddress = $"{employee.PostalCode}, {employee.Address}"
        let _salaryOutOk = decimal.TryParse(employee.Salary, out salary)
        select new
        {
            employee.ID,
            employee.Name,
            fullAddress,
            salary
        }
    ).ToList();
```

В коде выше понятно использование промежуточного преобразования с помощью оператора *let* для *fullAddress*. Для *salary* же происходит преобразование из строкового типа в *decimal*, причём результат преобразования попадает в *out* переменную *salary*, а переменной *_salaryOutOk* можно пренебречь.

Всё это делает код чище и яснее, иначе преобразование для *fullAddress* пришлось бы выносить прямо в проекцию *select*, а конструкцию *decimal.TryParse* вообще бы не удалось вынести в проекцию, так что альтернативы *let* нет.

Let в LINQ - это сила!
