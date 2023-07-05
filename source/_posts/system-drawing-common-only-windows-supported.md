---
title: System.Drawing.Common будет поддерживаться только на Windows
date: 2023-07-05 06:26:25
tags: [C#, .NET6]
---

Начиная с версии .NET6 и выше [Microsoft приостанавливает поддержку библиотеки System.Drawing.Common](https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/6.0/system-drawing-common-windows-only) на других платформах, кроме Windows. Это связано с рядом проблем кроссплатформенного портирования GDI+ на другие операционные системы (ОС). В вышеприведённой статье описаны альтернативы ```System.Drawing.Common```, но для версии .NET6 ещё можно продолжать использовать классы из ```System.Drawing.Common``` в отличных от Windows ОС.

Для этого:

1. В проекте, где используются классы из библиотеки ```System.Drawing.Common```, в файле проекта с расширением **.csproj** (обычно доступны по нажатию клавишы F4 из IDE) требуется добавить такую секцию:

``` csharp
  ...
  <ItemGroup>
    <RuntimeHostConfigurationOption Include="System.Drawing.EnableUnixSupport" Value="true" />
  </ItemGroup>
  ...
```

Это приведёт к тому, что при сборке проекта в файле вида ***[appname].runtimeconfig.json*** окажется секция:

``` json
{
   "runtimeOptions": {
      "configProperties": {
         "System.Drawing.EnableUnixSupport": true
      }
   }
}
```

2. Требуется убедиться, что на ОС установлен пакет **libgdiplus**. Если пакет отсутствует, то его нужно установить.
Для Linux это можно сделать так: ```sudo apt install libgdiplus```

Необходимо учесть, что начиная с .NET7 этот трюк больше не сработает, о чем предупреждается в вышеприведённой статье о Breaking Change от Microsoft.
