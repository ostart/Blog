---
title: Обязательный QueryString параметр в ASP.NET Core
date: 2022-12-21 13:43:32
tags: [C#, ASP.NETCore]
---

У меня возникла потребность для ```HttpGet``` запроса проверять, что в качестве ```QueryString``` параметра запроса прислан не просто какой-то там ```string```, а что присланое значение успешно преобразуется к перечислению ```Enum``` вида:

``` csharp
public enum QrCodeTypeEnum
{
    Location,
    Device 
}
```

Причём если ```QueryString``` параметр "левый", или вообще не прислан, то не должно самопроизвольно выбираться дефолтное значение перечисления ```QrCodeTypeEnum.Location``` как при обычном ```[FromQuery]``` аттрибуте (тут даже аттрибут ```[BindRequired]``` не поможет).

Для решения данной задачи потребовалось расширить возможности ```FromQueryAttribute``` и реализовать интерфейс ```IParameterModelConvention```, который обязует имплементировать метод ```void Apply(ParameterModel parameter)```.

Вот как я это сделал:
``` csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.AspNetCore.Mvc.ApplicationModels;

// ReSharper disable once CheckNamespace
namespace Microsoft.AspNetCore.Mvc
{
    public class RequiredFromQueryAttribute : FromQueryAttribute, IParameterModelConvention
    {
        public void Apply(ParameterModel parameter)
        {
            if (parameter.Action.Selectors != null && parameter.Action.Selectors.Any())
            {
                Func<string, bool> rejectionPredicate = null;
                
                if (parameter.ParameterType.IsEnum)
                {
                    var array = Enum.GetValues(parameter.ParameterType);
                    List<string> values = null;
                    foreach (var paramValue in array)
                    {
                        values ??= new List<string>();
                        values.Add(paramValue.ToString());
                    }

                    if (values is { Count: > 0 })
                        rejectionPredicate = x => !values.Contains(x, StringComparer.InvariantCultureIgnoreCase);
                }

                parameter.Action.Selectors.Last().ActionConstraints.Add(
                    new RequiredFromQueryActionConstraint(
                            parameter.BindingInfo?.BinderModelName ?? parameter.ParameterName, rejectionPredicate
                        )
                    );
            }
        }
    }
}
```

При инициализации контроллера, содержащего наш атрибут ```[RequiredFromQuery]```, происходит вызов метода ```Apply```, который понимает, что аттрибут применён к перечислению, благодаря свойству ```parameter.ParameterType.IsEnum```, и формирует предикат, который передаётся конструктору ```RequiredFromQueryActionConstraint``` и будет использован при поступлении запроса в контроллер для проверки поступающего значения в ```QueryString```.

Давайте взглянем как реализован класс ```RequiredFromQueryActionConstraint```, который имплементирует интерфейс ```IActionConstraint```:
``` csharp
using System;
using Microsoft.AspNetCore.Mvc.ActionConstraints;

// ReSharper disable once CheckNamespace
namespace Microsoft.AspNetCore.Mvc
{
    public class RequiredFromQueryActionConstraint : IActionConstraint
    {
        private readonly string _parameterKey;
        private readonly Func<string, bool> _rejectionPredicate;

        public RequiredFromQueryActionConstraint(string parameterKey, Func<string, bool> rejectionPredicate)
        {
            _parameterKey = parameterKey;
            _rejectionPredicate = rejectionPredicate;
        }

        public int Order => 999;

        public bool Accept(ActionConstraintContext context)
        {
            if (!context.RouteContext.HttpContext.Request.Query.ContainsKey(_parameterKey))
            {
                return false;
            }

            if (!context.RouteContext.HttpContext.Request.Query.TryGetValue(_parameterKey, out var value))
            {
                return false;
            }
            
            if (_rejectionPredicate != null && _rejectionPredicate.Invoke(value.ToString()))
            {
                return false;
            }

            return true;
        }
    }
}
```

При поступлении ```HttpGet``` запроса вида ```/v3/device/download-report?type=device``` к контроллеру ```device```, вызывается метод ```Accept``` класса ```RequiredFromQueryActionConstraint ```, который проверяет наличие ключа ```type```, наличие значения у этого ключа, и что значение ключа соответствует предикату ```_rejectionPredicate```, который формируется из значений перечисления в методе ```Apply``` атрибута ```RequiredFromQueryAttribute```. Теперь проверка ```QueryString``` параметра запроса осуществляется строго, и в случае отсутствия ключа ```type```, либо неверного (отсутствующего в перечислении) значения ключа ```type``` методом будет дан ответ с кодом 400 и сообщением вида:
``` json
{
    "id": [
        "The value 'download-report' is not valid."
    ]
}
```

Использовать атрибут ```[RequiredFromQuery]``` можно в методе контроллера следующим образом:
``` csharp
[HttpGet("download-report")]
[Authorize(Policy = PolicyNameProvider.AuthenticatedIdentityIsCurrentAccountAdmin)]
public FileStreamResult DownloadAccountQrCodesReport([RequiredFromQuery] QrCodeTypeEnum type)
{
    var qrCodeType = type.ToQrCodeType();
    ... и так далее
```

При решении данной задачи я опирался на статью [Required query string parameters in ASP.NET Core MVC](https://www.strathweb.com/2016/09/required-query-string-parameters-in-asp-net-core-mvc/), где содержится намного больше деталей описанного выше процесса.
