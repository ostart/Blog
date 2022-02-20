---
title: Имплементация интерфейса IAsyncDisposable
date: 2022-02-20 08:43:00
tags: [C#]
---

Интерфейс ```IAsyncDisposable``` служит для асинхронного освобождения неуправляемых ресурсов (unmanaged resources). Появился он в версии ```.NET Core 3.0```. В .NET классы, владеющие неуправляемыми ресурсами, обычно реализуют интерфейс ```IDisposable```, чтобы обеспечить механизм синхронного освобождения неуправляемых ресурсов (см. подробности в посте [Имплементация интерфейса IDisposable](https://ostart.github.io/2022/02/19/idisposable/)). Однако в некоторых случаях классам необходимо предоставлять асинхронный механизм освобождения неуправляемых ресурсов в дополнение к синхронному (или вместо него). Предоставление такого механизма позволяет потребителю выполнять ресурсоемкие операции удаления не блокируя основной поток приложения с графическим интерфейсом в течение длительного времени.

Метод ```IAsyncDisposable.DisposeAsync``` этого интерфейса возвращает ```ValueTask```, представляющий асинхронную операцию освобождения ресурсов. Класс, владеющий неуправляемыми ресурсами, реализует этот метод, а потребитель класса вызывает метод ```DisposeAsync``` объекта, когда он больше не нужен. Чтобы ресурсы освобождались даже в случае исключения следует поместить код, использующий объект IAsyncDisposable, в оператор using (в C#, начиная с версии 8.0), или вызвать метод DisposeAsync внутри finally оператора try/finally.

Приведём пример кода, поясняющий вышеописанное:
``` csharp
class SomeClass : IAsyncDisposable, IDisposable
{
    var asyncDisposeObj = new FileStream("SomeFile.txt", FileMode.OpenOrCreate, FileAccess.Write);
    var syncDisposeObj  = new HttpClient();

    // other usefull code

    public async ValueTask DisposeAsync()
    {
        await DisposeAsyncCore();

        Dispose(disposing: false);
        GC.SuppressFinalize(this);
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (disposing)
        {
            syncDisposeObj?.Dispose();
            (asyncDisposeObj as IDisposable)?.Dispose();
        }

        DisposeThisObject();
    }

    protected virtual async ValueTask DisposeAsyncCore()
    {
        if (asyncDisposeObj is not null)
        {
            await asyncDisposeObj.DisposeAsync().ConfigureAwait(false);
        }

        if (syncDisposeObj is IAsyncDisposable disposable)
        {
            await disposable.DisposeAsync().ConfigureAwait(false);
        }
        else
        {
            syncDisposeObj.Dispose();
        }

        DisposeThisObject();
    }

    void DisposeThisObject()
    {
        asyncDisposeObj = null;
        syncDisposeObj  = null;
    }
}
```

А где-то в приложении класс можно использовать так:
``` csharp
static async Task Main()
{
    await using var someClass = new SomeClass();
    // other usefull code
}
```

Код выше показал, в общих чертах, паттерн ```Async Dispose Pattern```. Как следует из кода, asyncDisposeObj ссылается на ресурс который следует утилизировать асинхронно, а syncDisposeObj - синхронно. Теперь синхронная и асинхронная утилизация ресурсов могут выполняться в одно время, так что важно рассматривать эти процессы совместно. Для синхронной и асинхронной утилизации ресурсов класс реализует интерфейсы ```IAsyncDisposable``` и ```IDisposable```. Как показано в посте [Имплементация интерфейса IDisposable](https://ostart.github.io/2022/02/19/idisposable/) ```IDisposable``` должен определять метод ```Dispose()``` без параметров, виртуальный метод ```Dispose(bool)``` и опциональный финализатор (деструктор). Наш пример не требует опционального финализатора (деструктора). А для ```IAsyncDisposable``` требуется определить методы без параметров ```DisposeAsync()``` и ```DisposeAsyncCore()```. Оба пути утилизации (синхронный и асинхронный) могут сработать, поэтому они оба должны быть готовы утилизировать ресурсы. В методе ```Dispose(bool)``` не только вызывается ```Dispose()``` для синхронного ресурса syncDisposeObj, но и произвоится попытка вызвать ```Dispose()``` для асинхронного asyncDisposeObj. Отметьте, что ```Dispose(bool)``` также вызывает ```DisposeThisObject```, который содержит тот же код, что и аналогичный код для асинхронного пути во избежание дублирования. 

Методы ```Dispose()``` и ```DisposeAsync()``` являются членами интерфейсов, а методы ```Dispose(bool)``` и ```DisposeAsyncCore()``` - конвенциями (условленными договоренностями). Оба последних метода виртуальные. Это является частью паттерна, когда производный класс может реализовать утилизацию ресурсов, переопределяя эти методы и вызывая их через ```base.Dispose(bool)``` и ```base.DisposeAsyncCore()```, чтобы гарантировать освобождение ресурсов по всей иерархии наследования.

Оба метода ```Dispose()``` и ```DisposeAsync()``` вызывают  ```Dispose(bool)```, но ```DisposeAsync()``` устанавливает флаг ```disposing``` в false. Напомню, что ```disposing = true``` это флаг утилизации управляемых ресурсов. Метод ```Dispose(bool)``` - это синхронный путь, а метод ```DisposeAsync()``` вызывает ```DisposeAsyncCore()``` для утилизирования асинхронных ресурсов. Как и ```Dispose(true)``` метод ```DisposeAsyncCore()``` пытается высвободить все управляемые ресурсы. Асинхронный случай очевиден, однако синхронный имеет пару особенностей. Что если синхронный объект сейчас или в будущем реализует ```IAsyncDisposable```? Тогда попытка вызова ```DisposeAsync()``` является наилучшим выбором в случае асинхронного пути выполнения кода. Иначе вызовется синхронный путь выполнения с методом ```Dispose()```.

В конце отметим, что метод ```Main``` использует конструкцию ```await using``` при создании экземпляра класса реализующего интерфейсы ```IAsyncDisposable``` и ```IDisposable```. Это гарантирует вызов метода ```DisposeAsync()``` по завершению выполнения метода ```Main```.