---
title: Разрабатываем Windows-службы с помощью TopShelf
date: 2021-09-30 16:05:13
tags: [C#, TopShelf, Windows-service]
---

Все, кто сталкивались с разработкой служб для операционной системы Windows, отмечали трудность отладки подобного ПО. Трудности старта разработки кода решаются шаблонами Visual Studio для Windows-служб, но вот сама отладка -- очень неудобная. Приходится подключаться к запущеному процессу через Debug --> Attach to Process... , выбирать процесс, ставить точку останова и т.д. Но всё изменилось когда появился [TopShelf](http://topshelf-project.com/). Этот фреймворк (*а именно так они себя величают*) позволяет разрабатывать и отлаживать Windows-службы как обычные консольные приложения!

Рассмотрим как разрабатывать Windows-службы с помощью [TopShelf](http://docs.topshelf-project.com/en/latest/).

Установить TopShelf в ваш проект можно через менеджер пакетов NuGet:
``` powershell
nuget Install-Package Topshelf
```

Посмотреть исходный код фреймворка **TopShelf** можно на [GitHub](https://github.com/Topshelf/Topshelf)

Основной класс (в нашем случае *AuthApiService*) должен содержать методы ``` public void Start() ``` и ``` public void Stop() ```, которые используются **TopShelf** при старте и остановке службы. В моём случае файл *Program.cs* же будет иметь следующий вид:
``` csharp
using Topshelf;

class Program
{
    private const string ServiceName = "Authentication and WebAPI service";

    static void Main(string[] args)
    {
        HostFactory.Run(config =>
        {
            config.StartAutomatically(); // startup type
            config.EnableShutdown();
            config.Service<AuthApiService>(service =>
            {
                service.ConstructUsing(svc => new AuthApiService());
                service.WhenStarted(svc => svc.Start());
                service.WhenStopped(svc => svc.Stop());
            });
            config.RunAsLocalSystem();
            config.SetServiceName(ServiceName);
            config.SetDisplayName(ServiceName);
            config.SetDescription(ServiceName);
            config.EnableServiceRecovery(service =>
            {
                service.OnCrashOnly();
                service.RestartService(delayInMinutes: 0); // first failure
                service.RestartService(delayInMinutes: 0); // second failure
                service.RestartService(delayInMinutes: 1); // subsequent failures
                service.SetResetPeriod(days: 1); // reset period for restart options
            });
        });                                          
    }
}
```

Всю магию делает метод *HostFactory.Run*. В нём указывается класс основного тела службы с методами *Start* и *Stop*, задаётся имя службы и описание в менеджере служб Windows (*Services*). **TopShelf** позволяет провести почти такую же настройку режима работы службы, которую выполняет менеджер служб Windows, в частности задать из под кого будет запускаться служба ``` RunAsLocalSystem ``` или задать параметры перезапуска в случае падения службы с помощью ``` EnableServiceRecovery ```.

Установить службу в операционную систему Windows можно из командной строки, запущенной от Администратора, прямо в папке где находится служба командой: 
```
KMAuthService.exe install
```

Удалить службу можно командой:
```
KMAuthService.exe uninstall
```

Всю магию установки и настройки службы берёт на себя **Topshelf**! 

Службу можно установить и самостоятельно, но тогда настройки из раздела ``` EnableServiceRecovery ``` применены не будут.
```
sc create "Authentication and WebAPI service" binPath="<путь_к_службе>\AuthService.exe"
```

Удалить службу самостоятельно можно командой:
```
sc delete "Authentication and WebAPI service"
```