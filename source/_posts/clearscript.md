---
title: ClearScript. Добавь скрипты в своё .NET приложение
date: 2021-09-07 16:14:06
tags: [C#, JavaScript, ClearScript, JS]
---

Были времена, когда обработку скриптов в .NET реализовывали проекты [IronPython](https://ironpython.net/) и [IronRuby](http://ironruby.net/). IronRuby уже умер, IronPython ещё жив, но надо признать, что не набрал популярности несмотря на солидный возраст. Если вам требуется использовать сценарии в своём .NET приложении обратите внимание на ClearScript.

[ClearScript](https://github.com/Microsoft/ClearScript) - это библиотека, которая с лёгкостью добавит сценарии на JavaScript (через V8 и JScript) или VBScript в ваше .NET приложение. ClearScript поддерживает несколько видов движков: Google's V8, Microsoft's JScript и VBScript. Посредством ClearScript можно запускать JavaScript сценарии как в старом CommonJS, так и в новой ES6 стандарте, причем сценарии будут работать как в Windows, Linux, так и macOS. Количество поддержанных фишек для разных движков несколько различается, но V8 содержит их в максимальном количестве. ClearScript доступен в виде NuGet пакетов для соответствующих платформ.

Для работы с ClearScript требуется подключить следующие пространства имён:

``` csharp
using Microsoft.ClearScript;
using Microsoft.ClearScript.JavaScript;
using Microsoft.ClearScript.V8;
```

Создать и инициализировать движок скриптов (если не требуется возможность отладки, то V8ScriptEngineFlags.EnableDebugging убрать, оставив просто **new()** ):

``` csharp
private static V8ScriptEngine Engine => new(V8ScriptEngineFlags.EnableDebugging)
{
    DocumentSettings =
    {
        AccessFlags = DocumentAccessFlags.EnableFileLoading,
        SearchPath = Path.GetDirectoryName(typeof(MyTests).Assembly.Location)
    }
};
```

Запуск вычислений в сценарии на JS приводится ниже:

``` csharp
var result = Engine.Evaluate(
    new DocumentInfo {Category = ModuleCategory.Standard}, 
    $"{_jsModuleContent} {PdpFunctionName}({JsonSerializer.Serialize(initialDevice)})"
);
var resultDevice = JsonSerializer.Deserialize<Device>(result.ToString());
```

*Category = ModuleCategory.Standard* означает, что текст сценария написан на ES6 (const, let, import, etc). *Category = ModuleCategory.CommonJS* - старый формат JS, понимающий только var и require. Из примера видно, что текстовый аргумент представляет из себя содержимое сценария и вызов JS функции с передачей ей в качестве аргумента сериализованного объекта JSON. В качестве результата возвращается экземпляр класса Object, который требуется десериализовать к нужному классу с помощью JsonSerializer.Deserialize. Таким образом, с помощью загрузки и исполнения JavaScript сценария можно динамически управлять выполняющимися инструкциями функции PdpFunctionName!

Удивительно, что можно подгружать классы .NET в движок ClearScript (в том числе даты, дженерики и LINQ) и использовать их для вычислений:

``` csharp
using (var engine = new V8ScriptEngine()) // создание движка
{
    engine.AddHostType("Console", typeof(Console)); // прокинуть тип в движок
    engine.Execute("Console.WriteLine('{0} is an interesting number.', Math.PI)"); // 3.14159265358979 is an interesting number.

    engine.AddHostObject("random", new Random()); // прокинуть объект в движок
    engine.Execute("Console.WriteLine(random.NextDouble())"); // 0.715555223503874

    engine.AddHostObject("lib", new HostTypeCollection("mscorlib", "System.Core")); // прокидываем целую сборку
    engine.Execute("Console.WriteLine(lib.System.DateTime.Now)"); // 5/11/2017 12:15:32 PM

    // создаем хост-объект прямо из скрипта
    engine.Execute(@"
        birthday = new lib.System.DateTime(2007, 5, 22);
        Console.WriteLine(birthday.ToLongDateString());
    "); // Tuesday, May 22, 2007

    // используем дженерик класс словаря прямо из скрипта
    engine.Execute(@"
        Dictionary = lib.System.Collections.Generic.Dictionary;
        dict = new Dictionary(lib.System.String, lib.System.Int32);
        dict.Add('foo', 123);
    ");

    engine.AddHostObject("host", new HostFunctions()); // вызываем хост-метод с out параметром
    engine.Execute(@"
        intVar = host.newVar(lib.System.Int32);
        found = dict.TryGetValue('foo', intVar.out);
        Console.WriteLine('{0} {1}', found, intVar);
    "); // True 123

    // создаем и перебираем хост массив
    engine.Execute(@"
        numbers = host.newArr(lib.System.Int32, 20);
        for (var i = 0; i < numbers.Length; i++) { numbers[i] = i; }
        Console.WriteLine(lib.System.String.Join(', ', numbers));
    "); // 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19

    // создаем в сценарии делегат
    engine.Execute(@"
        Filter = lib.System.Func(lib.System.Int32, lib.System.Boolean);
        oddFilter = new Filter(function(value) {
            return (value & 1) ? true : false;
        });
    ");

    // используем LINQ из скрипта
    engine.Execute(@"
        oddNumbers = numbers.Where(oddFilter);
        Console.WriteLine(lib.System.String.Join(', ', oddNumbers));
    "); // 1, 3, 5, 7, 9, 11, 13, 15, 17, 19

    // используем динамический хост объект
    engine.Execute(@"
        expando = new lib.System.Dynamic.ExpandoObject();
        expando.foo = 123;
        expando.bar = 'qux';
        delete expando.foo;
    ");

    engine.Execute("function print(x) { Console.WriteLine(x); }"); // создаем и затем вызываем функцию из скрипта
    engine.Script.print(DateTime.Now.DayOfWeek); // Thursday

    engine.Execute("person = { name: 'Fred', age: 5 }");  // создадим из скрипта объект
    Console.WriteLine(engine.Script.person.name); // Fred

    engine.Execute("values = new Int32Array([1, 2, 3, 4, 5])"); // создадим в скрипте типизированный массив
    var values = (ITypedArray<int>)engine.Script.values; // считаем и преобразуем из Object значения массива
    Console.WriteLine(string.Join(", ", values.ToArray())); // 1, 2, 3, 4, 5
}
```

Отличный [ClearScript Tutorial](https://microsoft.github.io/ClearScript/Tutorial/FAQtorial.html)