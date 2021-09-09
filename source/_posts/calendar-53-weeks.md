---
title: Почему в календаре бывает 53 недели
date: 2021-09-09 13:00:16
tags: [C#, Calendar, ISOWeek, GetWeekOfYear]
---

Согласно ГОСТ ИСО 8601-2001 "ПРЕДСТАВЛЕНИЕ ДАТ И ВРЕМЕНИ" расчет номера недели в Российской Федерации осуществляется так:

*Первой неделей года является та неделя, которая содержит число 4 января. То есть если первое января выпадает на пятницу, субботу, или воскресенье, то все еще продолжается последняя неделя предыдущего года.*

Это может приводить к интересным спецэффектам - в году может содержаться 52 или 53 недели, 29, 30, 31 декабря могут относиться к первой неделе следующего года, и наоборот, 1, 2, и 3 января могут относиться к 52 или 53 неделе прошлого года. В частности 53 недели содержали **2015** и **2020** годы.

Привычный подход к вычислению количества недель в году с помощью Calendar.GetWeekOfYear для нашего ГОСТа даёт сбой и **работает некорректно**. 53-и недели при таком подходе отсустсвуют.

``` csharp
// Такой подход вычисляет 53-ю неделю НЕКОРРЕКТНО
var ruCi = new CultureInfo("ru-RU");
var ruCal = ruCi.Calendar;
var ruCwr = ruCi.DateTimeFormat.CalendarWeekRule;
var ruFirstDow = ruCi.DateTimeFormat.FirstDayOfWeek;

for (var i = 1; i < dateTimeRanges.Count; i++)
{
    var year = dateTimeRanges[i].Add(reportDto.ClientOffset).Year.ToString("D4");
    var nn = ruCal.GetWeekOfYear(
        dateTimeRanges[i].Add(reportDto.ClientOffset), 
        ruCwr, 
        ruFirstDow
    ).ToString("D2"); // вычисляет 53-ю неделю НЕКОРРЕКТНО !!!
    worksheet.Cells[4, shift + i].Value = $"{nn}.{year}"; // НЕКОРРЕКТНОЕ значение для 53-ей недели!!!
}
```

**Метод Calendar.GetWeekOfYear вычиляет недели в году не по ГОСТ ИСО 8601-2001 и для календаря РФ не подходит!**

Для расчёта недель согласно ГОСТ ИСО 8601-2001 следует использовать статический класс **ISOWeek** и метод **GetWeekOfYear**:

``` csharp
for (var i = 1; i < dateTimeRanges.Count; i++)
{
    var year = ISOWeek.GetYear(dateTimeRanges[i].Add(reportDto.ClientOffset)).ToString("D4");
    var nn = ISOWeek.GetWeekOfYear(
        dateTimeRanges[i].Add(reportDto.ClientOffset)
    ).ToString("D2"); // ПРАВИЛЬНОЕ значение для 53-ей недели
    worksheet.Cells[4, shift + i].Value = $"{nn}.{year}";
}
```

Подробнее с **ISOWeek.GetWeekOfYear** можно познакомиться [по ссылке](https://docs.microsoft.com/ru-ru/dotnet/api/system.globalization.isoweek.getweekofyear?view=net-5.0)