---
title: Добавляем WebAPI в Windows-службу
date: 2021-10-07 14:09:48
tags: [C#, OWIN, WebAPI, Windows-service]
---

В предыдущей статье мы рассмотрели [создание Windows-службы с помощью фреймворка TopShelf](https://ostart.github.io/2021/09/30/topshelf/). Но затем мне поднадобилось добавить возможности WebAPI в нашу службу. Рассмотрим как это можно сделать. 

Тут нам поможет *OWIN*. С помощью **NuGet Package Manager** устанавливаем пакет в наш проект службы:
``` powershell
nuget Install-Package Microsoft.AspNet.WebApi.OwinSelfHost 
```

Затем конфигурируем наш self-host WebAPI путём создания *Startup.cs* файла следующего вида:
``` csharp
using System.Net.Http.Formatting;
using Owin;
using System.Web.Http;
using System.Web.Http.Cors;
using Newtonsoft.Json;
using Newtonsoft.Json.Converters;
using Newtonsoft.Json.Serialization;

public class Startup
{
    // Don't delete! This code configures Web API. 
    // The Startup class is specified as a type parameter in the WebApp.Start method
    public void Configuration(IAppBuilder appBuilder)
    {
        var config = new HttpConfiguration(); // Configure Web API for self-host
        config.Routes.MapHttpRoute(
            name: "DefaultApi",
            routeTemplate: "api/{controller}/{id}",
            defaults: new { id = RouteParameter.Optional }
        );
        config.Formatters.Clear();
        config.Formatters.Add(new JsonMediaTypeFormatter());
        config.Formatters.JsonFormatter.SerializerSettings =
            new JsonSerializerSettings
            {
                ContractResolver = new CamelCasePropertyNamesContractResolver()
            };
        config.Formatters.JsonFormatter.SerializerSettings.Converters.Add(new StringEnumConverter());
        var cors = new EnableCorsAttribute("*", "*", "*");
        config.EnableCors(cors);
        appBuilder.UseWebApi(config);
    }
}
```

Выше происходит настройка роутинга, JSON формата запроса/ответа и cors.

Далее в проект требуется добавить WebAPI контроллер. Для примера возьмём контроллер из [оригинальной статьи на сайте Microsoft](https://docs.microsoft.com/en-us/aspnet/web-api/overview/hosting-aspnet-web-api/use-owin-to-self-host-web-api), т.к. используемые мною контроллеры только загромоздят суть повествования излишним содержимым:
``` csharp
using System.Collections.Generic;
using System.Web.Http;

public class ValuesController : ApiController 
{ 
    // GET api/values 
    public IEnumerable<string> Get() 
    { 
        return new string[] { "value1", "value2" }; 
    } 

    // GET api/values/5 
    public string Get(int id) 
    { 
        return "value"; 
    } 

    // POST api/values 
    public void Post([FromBody]string value) 
    { 
    } 

    // PUT api/values/5 
    public void Put(int id, [FromBody]string value) 
    { 
    } 

    // DELETE api/values/5 
    public void Delete(int id) 
    { 
    } 
}
```

Теперь отредактируем Program.cs файл Windows-службы для одновременного запуска с WebAPI:
``` csharp
using System;
using Topshelf;
using Microsoft.Owin.Hosting;
using System.Configuration;

public class Program 
{ 
    private const string ServiceName = "Authentication and WebAPI service";

    static void Main() 
    { 
        var port = ConfigurationManager.AppSettings["Port"];

        var options = new StartOptions();
        options.Urls.Add($"http://*:{port}/");

        using (WebApp.Start<Startup>(options)) // Start OWIN host 
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
} 
```

В результате WebAPI начинает прослушивать все входящие запросы на порт, считанный из файла конфигурации и работает Windows-служба, разработанная с помощью фреймворка Topshelf. Вот так вота! Или как говорят французы: Вуаля!