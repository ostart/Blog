---
title: Особенность отображения Swagger'ом XML комментариев
date: 2022-02-24 10:15:31
tags: [C#, Swagger]
---

[Swashbuckle Swagger](https://swagger.io/) – это фреймворк для спецификаций RESTful API, которые выраженны с помощью JSON. Swagger используется вместе с набором программных инструментов с открытым исходным кодом для проектирования, создания, документирования и использования веб-служб RESTful. Его сильная сторона заключается в том, что он дает возможность не только интерактивно просматривать спецификацию контроллеров WebAPI, но и отправлять запросы через Swagger UI.

Swagger – удобный инструмент для реализации встроенной в код документации. Добавление XML-комментариев методам контроллеров делает описание методов наглядными при работе через Swagger UI. Отличное руководство по [Началу работы с Swashbuckle и ASP.NET Core](https://docs.microsoft.com/ru-ru/aspnet/core/tutorials/getting-started-with-swashbuckle?view=aspnetcore-6.0&tabs=visual-studio) расположено на сайте Майкрософт. Там приведены рекомендации по подключению Swagger и по оформлению XML-комментариев к методам контроллеров WebAPI.

Рассмотрим пример одного из таких XML-комментариев метода ```Create```:

``` csharp
/// <summary>
/// Creates a TodoItem.
/// </summary>
/// <param name="item"></param>
/// <returns>A newly created TodoItem</returns>
/// <remarks>
/// Sample request:
///
///     POST /Todo
///     {
///        "id": 1,
///        "name": "Item #1",
///        "isComplete": true
///     }
///
/// </remarks>
/// <response code="201">Returns the newly created item</response>
/// <response code="400">If the item is null</response>
[HttpPost]
[ProducesResponseType(StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<IActionResult> Create(TodoItem item)
{
    _context.TodoItems.Add(item);
    await _context.SaveChangesAsync();

    return CreatedAtAction(nameof(Get), new { id = item.Id }, item);
}
```

Обратите внимание, как эти дополнительные комментарии улучшают пользовательский интерфейс:

{% asset_img swagger-post.png Swagger %}

Однако я столкнулся со следующей проблемой: в случае присутствия одного из следующих символов: ```& > < “ ‘``` в XML-комментарии, описание метода в Swagger UI полностью ломается и перестаёт отображаться, т.е. метод начинает выглядеть так, как будто никакого XML-комментария вовсе и нет.

Проблема заключается в природе самого XML, который по своей сути знает только об ```&amp; &lt; &gt; &apos; &quot;```, а также числовых объектах. Поэтому вместо ```& > < ‘ “``` нужно использовать ```&amp; &lt; &gt; &apos; &quot;```. В таком случае комментарии к методам WebAPI  исчезать из Swagger UI не будут.
