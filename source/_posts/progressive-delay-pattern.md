---
title: Шаблон увеличивающегося интервала повторения запроса
date: 2021-11-18 09:04:13
tags: [C#, ProgressiveDelayPattern]
---

Когда в приложении прогнозируется присутствие проблем при запросах по сети и не требуется сдаваться в получении ответа при появлении первой же проблемы, я использую шаблон увеличивающегося интервала повторения запроса. Суть шаблона такова: в случае отсутствия ответа (от МФУ например) в отведённый таймаут приложение ожидает определённый интервал времени и затем пытается отправить запрос снова. Если и во второй раз нет ответа, то ожидаем в два раза дольший интервал времени и т.д. (в три, четыре... Вид прогрессии можно выбрать самостоятельно). Получается, что запросы идут всё реже и реже не перегружая устройство (или сайт) в случае сетевых проблем на его стороне. В случае отсутствия ответов заданное число раз, можно полностью прекратить отправку запросов.

Ниже приводится код условного метода *Method*. База интервала повторения запроса составляет одну секунду и задана арифметическая прогрессия увеличения интервала. Можно задать геометрическую прогрессию, умножая текущий интервал повторения на 2 каждый раз или даже степенную, возводя базу в степень текущего числа повторений (```var currentDelay = Math.Pow(DelayInMilliseconds, tryCounter);```). Ожидание реализовано синхронно, а асинхронный вариант ожидания (требующий изменения сигнатуры метода на *Task<int> Method()*) показан в комментарии.

```csharp
public static int Method() // public static Task<int> Method()
{
    const int DelayInMilliseconds = 1000;
    const int RetryCount = 3;

    private bool success = false;
    private int tryCounter = 0;
    private int response;

    try
    {
      do
      {
          try
          {
              response = // Http запрос к сайту, устройству или т.п.
              success = true;
          } 
          catch(HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.RequestTimeout)
          {
              tryCounter += 1;
              var currentDelay = DelayInMilliseconds * tryCounter;
              Thread.Sleep(currentDelay); // await Task.Delay(currentDelay);
          }
      } while(tryCounter < RetryCount);
    }
    finally
    {
      if (success)
        return response;
      else
        throw new HttpRequestException("Request failed");
    }
}
```

Обратите внимание, на двойное обёртывание блоками try-catch-finally самого сетевого запроса. Именно во внешнем блоке finally формируется возвращаемый методом ответ ```return response;```. Иначе он выкидывает наружу исключение HttpRequestException.