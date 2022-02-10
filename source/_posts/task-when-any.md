---
title: Параллельная обработка задач по мере их завершения с помощью Task.WhenAny
date: 2022-02-10 12:42:02
tags: [C#]
---

Читаю я как-то главу **6.8 "Handling Parallel Tasks as They Complete"** книги **Joe Mayo "C# Cookbook"** и встречаю такой абзац:

*You thought calling* ```Task.WhenAny``` *would be an efficient use resources for processing results as they complete, but cost and performance are terrible.*

Минуточку, уважаемый Джо! А тут я попрошу вас поподробнее. У меня на проде используется этот ```Task.WhenAny```... Но в подробности Джо не пускается, рекомендуя вместо ~```Task.WhenAny```~ использовать ```Task.WhenAll```, утверждая, что у ```Task.WhenAll``` сложность О(1), т.к. вместо N последовательных операций выполняется как-бы одна (все N операций выполняются параллельно).

Далее Джо утверждает, что в паттерне:
``` csharp
while(tasks.Any())
{
    var currentFinishedTask = await Task.WhenAny(tasks);
    tasks.Remove(currentFinishedTask);
    // Handle currentFinishedTask result
}
```

всё происходит не так как ты ожидаешь:

*The reality is that each subsequent loop starts a brand-new set of tasks. This is how async works - you can* ```await``` *a task multiple times, but each* ```await``` *starts a new task. That means the code continiously starts new instances of remaining tasks on every loop. This looping pattern, with* ```Task.WhenAny```*, doesn't result in the O(1) performance you might have expected, like with* ```Task.WhenAll```*, but rather O(N^2).*

А это уже плохой сигнал, как говорит [Егор Иванов](https://www.youtube.com/channel/UC3VDbjQ1hnY4zSRHgAfAK-A) :( Далее Джо пугает, что это скажется не только на плохой производительности приложения, но и может породить избыточную сетевую активность и нагрузить конечный сервер бесполезной работой. А представляете как эти избыточные сетевые запросы отразятся и сколько будут стоить в облачном сервисе?!? Короче Джо признаёт ```Task.WhenAny``` антипаттерном в том виде, что он приведён выше, если он только не используется для малого числа задач. Джо советует использовать ```Task.WhenAny``` только для случая, когда нас интересует только задача победитель, а на остальные можно забить и не дожидаться/продолжать их обработку. Тогда ```Task.WhenAny``` тоже будет иметь производительность со сложностью O(1).

Ну и завершает главу Джо таким абзацем:
*Task.WhenAny runs those tasks in parallel and the first task to complete comes back. The code processes that task and returns. The side effect in this solution is that tasks other the first that returned continue running. However, you don't have access to them because only one task is returned. For this scenario, you don't care about those tasks but should stop them to avoid using more resources than necessary. You can learn how to do that in the next section on cancelling tasks.*

Вот такие пироги! Побочный эффект, понимаешь! Сейчас проверим, что за побочный эффект. Для этого наваяем такой класс:

``` csharp
public class ExperimentalTask
{
    public int TimeInSeconds {get; private set;}
    public ExperimentalTask(int timeInSeconds)
    {
        TimeInSeconds = timeInSeconds;
    }

    public async Task WaitFor()
    {
        Console.WriteLine($"Enter in Time={TimeInSeconds} sec.");
        await Task.Delay(TimeInSeconds * 1000);
        Console.WriteLine($"Finished for Time={TimeInSeconds} sec.");
    }
}
```

и задействуем его таким кодом:

``` csharp
Console.WriteLine("Experiment is starting");

var tasks = new List<Task>();

for(var i = 1; i <= 5; i += 1)
{
    tasks.Add(new ExperimentalTask(i).WaitFor());
}

var stopwatch = System.Diagnostics.Stopwatch.StartNew();

while(tasks.Any())
{
    Console.WriteLine($"tasks.Count={tasks.Count}");
    var currentFinishedTask = await Task.WhenAny(tasks);
    tasks.Remove(currentFinishedTask);
}

Console.WriteLine("Experiment finished");
stopwatch.Stop();
Console.WriteLine($"Elapsed time: {stopwatch.Elapsed}\n");
```

Запустим приложение и увидим такие результаты:

``` bash
Experiment is starting
Enter in Time=1 sec.
Enter in Time=2 sec.
Enter in Time=3 sec.
Enter in Time=4 sec.
Enter in Time=5 sec.
tasks.Count=5
Finished for Time=1 sec.
tasks.Count=4
Finished for Time=2 sec.
tasks.Count=3
Finished for Time=3 sec.
tasks.Count=2
Finished for Time=4 sec.
tasks.Count=1
Finished for Time=5 sec.
Experiment finished
Elapsed time: 00:00:05.0092732
```

Так, а где побочный эффект? Где "the code continiously starts new instances of remaining tasks on every loop"? Про какую реальность утверждает Джо "The reality is that each subsequent loop starts a brand-new set of tasks"? При исследовании Task.WhenAny паттерна на сайте Майкрософт [Обработка асинхронных задач по мере завершения (C#)](https://docs.microsoft.com/ru-ru/dotnet/csharp/programming-guide/concepts/async/start-multiple-async-tasks-and-process-them-as-they-complete) находим примечание:

**Внимание!**

**Можно использовать** ```WhenAny``` **в цикле, как описано в примере, для решения проблем, которые включают небольшое число задач. Однако когда требуется обработка большого числа задач, другие методы будут более эффективны. Дополнительные сведения и примеры см. в разделе** [Обработка задач по мере их завершения](https://devblogs.microsoft.com/pfxteam/processing-tasks-as-they-complete/).

Предлагаю углубиться в эту тему через мой [перевод этой статьи](https://ostart.github.io/2022/02/10/processing-tasks-as-they-complete/).