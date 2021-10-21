---
title: Пагинация с помощью курсора в Entity Framework Core и ASP.NET Core
date: 2021-10-21 10:59:07
tags: [C#, EntityFramework, paging, cursor]
---

**Перевод статьи Халида Абухакмеха (Khalid Abuhakmeh) [Cursor Paging With Entity Framework Core and ASP.NET Core](https://khalidabuhakmeh.com/cursor-paging-with-entity-framework-core-and-aspnet-core)**.

При создании API-интерфейсов, которые возвращают значительный объем данных, нам необходимо принимать проектные решения, основываясь на нижележащей базе данных. Например, разработчик может реализовать специальный endpoint в API, который позволит клиентам получать наборы значений из заданного диапазона. Как разработчики API, мы можем выбирать между постраничной пагинацией и пагинацией с помощью курсора.

В этом посте показано, как реализовать оба подхода и почему разбиение по страницам с помощью курсора - это более предпочтительный по производительности вариант.

## Что такое пагинация?

Для разработчиков, только начинающих создавать управляемые данными API, понятие пагинации, как разбиения на страницы, становится важной для понимания концепцией. Набор значений может быть как ограниченным, так и безграничным.

Ограниченный набор значений имеет предел. Например, семья будет иметь верхнюю границу количества детей. В большинстве случаев в семье бывает 2-4 ребенка. Таким образом, мы, скорее всего, вернем всех детей в едином ответе при построении API на базе семьи.

Безграничный набор значений не имеет предела. Обычно это коллекции, связанные с вводом пользователя, данными временных рядов или другими механизмами. Например, все платформы социальных сетей работают с безграничным ресурсом. Например, в Твиттере твиты отправляют миллионы людей по всей планете одновременно. В этих случаях невозможно вернуть полный набор данных любому клиенту, но вместо этого API создает фрагмент на основе запрашиваемового критерия.

Что касается API, то пагинация - это попытка разработчика предоставить клиенту частичный набор результатов из огромного источника данных, с которым невозможно взаимодействовать как то иначе.

## Выбор правильного подхода к пагинации

Существует два подхода к разбиению коллекций на страницы:

1. Пагинация по номеру и размеру страницы
2. Пагинация с использованием курсора и размера страницы

Термин **курсор** переопределён. Его не следует путать с понятием курсора из реляционных баз данных. Хотя идея имеет сходство с реализацией в базе данных, они не связаны между собой.

**Курсор** - это идентификатор, который извлекает следующий элемент в наших последовательных запросах пагинации. Таким образом, пользователи могут размышлять об этом отвечая на вопрос: «Что последует за этим идентификатором курсора?»

Давайте рассмотрим на два подхода в формате запроса:
```
### Пагинация по номеру и размеру страницы
GET https://localhost:5001/pictures/paging?page=100000&size=10
Accept: application/json

### Пагинация с использованием курсора и размера страницы
GET https://localhost:5001/pictures/cursor?after=999990&size=10
Accept: application/json
```

Два HTTP-запроса выглядят очень похоже с точки зрения пользователя, но работают принципиально по-разному.

Для запроса пагинации мы вычисляем диапазон в нашем хранилище данных и извлекаем эти элементы. Здесь мы разместим коллекцию изображений, чтобы получить результаты на определенной странице:
``` csharp
var query = db
    .Pictures
    .OrderBy(x => x.Id)
    .Skip((page - 1) * size)
    .Take(size);
```

Хотя данный подход работает, разработчики должны помнить о предостережениях, самым большим из которых является смещение диапазона по мере поступления новых данных. Другая проблема в данном подходе зависит от нижележащей базы данных. При разбиении на страницы с помощью диапазонов база данных должна сканировать всё множество результатов, чтобы вернуть клиенту страницу. Таким образом, когда мы перемещаемся всё ближе к концу коллекции, производительность начинает ухудшаться.

А как работает пагинация с помощью курсора? В отличие от создания диапазона для получения множества результатов, мы используем предыдущий результат для получения следующего. Курсором может быть любое значение, гарантирующее следующий диапазон. Поля, такие как авто-инкрементируемые идентификаторы или отметки времени, идеально подходят для пагинации с помощью курсора. Пагинация с помощью  курсора не страдает от проблемы, связанной с тем, что входящие данные искажают наши результаты разбиения по страницам, т.к. наш порядок является детерминированным.

Давайте рассмотрим, как реализовать пагинацию с помощью курсора в виде запроса LINQ:
``` csharp
var query = db
    .Pictures
    .OrderBy(x => x.Id)
    // будет задействован индекс
    .Where(x => x.Id > after)
    .Take(size);
```

Пагинация достаточно дорогостоящая операция, какой бы подход мы не использовали. Вы когда-нибудь задумывались, почему Google перестает показывать страницы результатов поиска после максимум 25 страниц? Что ж, существует способ уменьшения этой стоимости и повышения производительности.

## Сравнивая производительность

Один из наиболее заметных недостатков пагинации по номеру и размеру страницы по сравнению с пагинацией с помощью курсора - это влияние на производительность. Давайте сравним две реализации веб-приложения в ASP.NET Core.

**Примечание: В этом примере используются минималистические ASP.NET Core API-интерфейсы.**

``` csharp
using System.Linq;
using CursorPaging;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<Database>();

var app = builder.Build();
await using (var scope = app.Services.CreateAsyncScope())
{
    var db = scope.ServiceProvider.GetService<Database>();
    var logger = scope.ServiceProvider.GetService<ILogger<WebApplication>>();
    var result = await Database.SeedPictures(db);
    logger.LogInformation($"Seed operation returned with code {result}");
}

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app
    .UseDefaultFiles()
    .UseStaticFiles();

app.MapGet("/pictures/paging", async http =>
{
    var page = http.Request.Query.TryGetValue("page", out var pages)
        ? int.Parse(pages.FirstOrDefault() ?? string.Empty)
        : 1;

    var size = http.Request.Query.TryGetValue("size", out var sizes)
        ? int.Parse(sizes.FirstOrDefault() ?? string.Empty)
        : 10;

    await using var db = http.RequestServices.GetRequiredService<Database>();
    var total = await db.Pictures.CountAsync();
    var query = db
        .Pictures
        .OrderBy(x => x.Id)
        .Skip((page - 1) * size)
        .Take(size);

    var logger = http.RequestServices.GetRequiredService<ILogger<Database>>();
    logger.LogInformation($"Using Paging:\n{query.ToQueryString()}");

    var results = await query.ToListAsync();
    await http.Response.WriteAsJsonAsync(new PagingResult
    {
        Page = page,
        Size = size,
        Pictures = results.ToList(),
        TotalCount = total,
        Sql = query.ToQueryString()
    });
});

app.MapGet("/pictures/cursor", async http =>
{
    var after = http.Request.Query.TryGetValue("after", out var afters)
        ? int.Parse(afters.FirstOrDefault() ?? string.Empty)
        : 0;

    var size = http.Request.Query.TryGetValue("size", out var sizes)
        ? int.Parse(sizes.FirstOrDefault() ?? string.Empty)
        : 10;

    await using var db = http.RequestServices.GetRequiredService<Database>();
    var logger = http.RequestServices.GetRequiredService<ILogger<Database>>();

    var total = await db.Pictures.CountAsync();
    var query = db
        .Pictures
        .OrderBy(x => x.Id)
        // will use the index
        .Where(x => x.Id > after)
        .Take(size);

    logger.LogInformation($"Using Cursor:\n{query.ToQueryString()}");

    var results = await query.ToListAsync();

    await http.Response.WriteAsJsonAsync(new CursorResult
    {
        TotalCount = total,
        Pictures = results,
        Cursor = new CursorResult.CursorItems
        {
            After = results.Select(x => (int?) x.Id).LastOrDefault(),
            Before = results.Select(x => (int?) x.Id).FirstOrDefault()
        },
        Sql = query.ToQueryString()
    });
});

app.Run();
```

Разместим в нашей базе данных SQLite миллион строк и попытаемся постранично продвинуться по коллекции как можно глубже.

```
### Пагинация по номеру и размеру страницы
GET https://localhost:5001/pictures/paging?page=100000&size=10
Accept: application/json

### Пагинация с использованием курсора и размера страницы
GET https://localhost:5001/pictures/cursor?after=999990&size=10
Accept: application/json
```

Каковы результаты этого эксперимента?

Для нашего запроса страница/размер мы видим общее время ответа 225 мс.

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Wed, 23 Jun 2021 12:46:27 GMT
Server: Kestrel
Transfer-Encoding: chunked

> 2021-06-23T084627.200.json

Response code: 200 (OK); Time: 225ms; Content length: 1328 bytes
```

Посмотрим, как курсор поведёт себя в подобной ситуации?

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Wed, 23 Jun 2021 12:46:27 GMT
Server: Kestrel
Transfer-Encoding: chunked

> 2021-06-23T084627-1.200.json

Response code: 200 (OK); Time: 52ms; Content length: 1370 bytes
```

Ого! 52 мс против 225 мс! Это существенная разница при том же результате. Откуда же возникло это улучшение производительности? Давайте рассмотрим SQL запросы, сгенерированные EF Core, и сравним их.

``` sql
-- Использование страницы/размера
.param set @__p_1 10
.param set @__p_0 999990

SELECT "p"."Id", "p"."Created", "p"."Url"
FROM "Pictures" AS "p"
ORDER BY "p"."Id"
LIMIT @__p_1 OFFSET @__p_0

-- Использование курсора
.param set @__after_0 999990
.param set @__p_1 10

SELECT "p"."Id", "p"."Created", "p"."Url"
FROM "Pictures" AS "p"
WHERE "p"."Id" > @__after_0
ORDER BY "p"."Id"
LIMIT @__p_1
```

Видно, что первый запрос (страница/размер) использует ключевое слово OFFSET. Это ключевое слово говорит нашему провайдеру базы данных SQLite сканировать до смещения перед вызовом ключевого слова LIMIT. Как можно себе представить, сканирование таблицы из миллиона строк достаточно дорогостоящая операция.

Для запроса с курсором видно, что используется столбец Id с условием WHERE. Использование индекса позволяет SQLite эффективно перемещаться по таблице, обеспечивая и более эффективное выполнение запроса.

## Заключение

Как мы видели в примерах выше, пагинация с помощью курсора становится более производительной по мере увеличения объёма данных. Тем не менее, пагинация с использованием страницы и размера может имееть свое место в приложениях. Этот подход намного проще реализовать и для небольших наборов данных, которые меняются нечасто, это может быть отличным выбором. Как разработчики, мы обязаны выбрать лучший вариант для нашего сценария. Тем не менее, учитывая почти четырехкратное улучшение производительности запросов, трудно поспорить с пагинацией с помощью курсора.