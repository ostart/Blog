---
title: Особенности потоко-безопасного использования класса HttpClient при отправке Http Headers
date: 2021-08-22 08:23:44
tags: [C#, HttpClient, Headers, DefaultRequestHeaders, thread-safe, concurrency]
---

Разрабатывая Windows-службу, работающую с МФУ и принтерами по HTTP, я использовал класс HttpClient. Данный класс является потоко-безопасным и не требует создания многочисленных экземпляров: достаточно один раз создать HttpClient, чтобы он обсужил все требующиеся запросы (в том числе многопоточные):

``` csharp
private static readonly HttpClient Client = new HttpClient();
```

или если не требуется, чтобы использовались Cookies:

``` csharp
private static readonly HttpClient Client = new HttpClient(new HttpClientHandler { UseCookies = false });
```

Данный подход избавит от проблемной утилизации (Dispose) экземпляров HttpClient и потенциального исчерпания свободных портов в системе (port exhausting), связанного с проблемной утилизацией.

Всё бы хорошо, но мною была обнаружена **проблема разделяемого общего состояния эксземпляра класса HttpClient**, когда мне потребовалось кроме сообщений ещё и **отправлять Http Headers с аутентификацией** и служебной информацией.

Оказалось, что мною используемый подход:

``` csharp
Client.DefaultRequestHeaders.Clear(); // doesn't work for concurrency
Client.DefaultRequestHeaders.Add("Authorization", token); // doesn't work for concurrency
var response = await Client.GetAsync(requestUri);
response.EnsureSuccessStatusCode();

var result = await response.Content.ReadAsStringAsync();
return JsonConvert.DeserializeObject<UserProfile>(result);
```

некореектно работает в многопоточной среде, т.к. имеет разделяемое общее состояние хэдеров Http Headers. 

Исследование проблемы натолкнуло меня на отличную статью с решением данной проблемы: [Concurrency with HttpClient](https://www.thinkprogramming.co.uk/concurrency-with-httpclient/)

Решение заключается в отказе от внутреннего разделяемого общего состояния HttpClient'а, а именно от Default Header. Следует дописать метод расширения, который делает то же самое, что и вышеобозначенный код, но без разделяемого общего состояния:

``` csharp
public static class HttpClientExtensions
{
    public static async Task<HttpResponseMessage> GetAsyncExt(
        this HttpClient @this,
        string url,
        Action<HttpRequestHeaders> beforeRequest)
    {
        var request = new HttpRequestMessage(HttpMethod.Get, url);
        beforeRequest(request.Headers);
        return await @this.SendAsync(request);
    }

    public static async Task<HttpResponseMessage> PostAsyncExt(
        this HttpClient @this,
        string url,
        HttpContent content,
        Action<HttpRequestHeaders> beforeRequest)
    {
        var request = new HttpRequestMessage(HttpMethod.Post, url);
        beforeRequest(request.Headers);
        request.Content = content;
        return await @this.SendAsync(request);
    }
}
```

Теперь вызовы из кода HttpClient с передачей хэдеров Http Headers выглядят так и являются потоко-безопасными:

``` csharp
var response = await Client.GetAsyncExt(requestUri, h =>
{
    h.Add("Authorization", token);
});
response.EnsureSuccessStatusCode();

var result = await response.Content.ReadAsStringAsync();
return JsonConvert.DeserializeObject<UserProfile>(result);
```


