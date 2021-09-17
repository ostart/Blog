---
title: Добавляем из терминала VSCode NuGet пакеты в C# проект
date: 2021-09-17 13:24:36
tags: [C#, NuGet, VSCode, net5.0]
---

Существует возможность создания разнообразных видов проектов ***.NET 5*** прямо из терминала ***VSCode***. Взгляните:

```
D:\Github\EnumSharp>dotnet new 
Template Name                                 Short Name           Language    Tags
--------------------------------------------  -------------------  ----------  ----------------------
Console Application                           console              [C#],F#,VB  Common/Console        
Class library                                 classlib             [C#],F#,VB  Common/Library        
WPF Application                               wpf                  [C#],VB     Common/WPF
WPF Class library                             wpflib               [C#],VB     Common/WPF
WPF Custom Control Library                    wpfcustomcontrollib  [C#],VB     Common/WPF
WPF User Control Library                      wpfusercontrollib    [C#],VB     Common/WPF
Windows Forms App                             winforms             [C#],VB     Common/WinForms       
Windows Forms Control Library                 winformscontrollib   [C#],VB     Common/WinForms       
Windows Forms Class Library                   winformslib          [C#],VB     Common/WinForms       
Worker Service                                worker               [C#],F#     Common/Worker/Web     
MSTest Test Project                           mstest               [C#],F#,VB  Test/MSTest
NUnit 3 Test Project                          nunit                [C#],F#,VB  Test/NUnit
NUnit 3 Test Item                             nunit-test           [C#],F#,VB  Test/NUnit
xUnit Test Project                            xunit                [C#],F#,VB  Test/xUnit
Razor Component                               razorcomponent       [C#]        Web/ASP.NET
Razor Page                                    page                 [C#]        Web/ASP.NET
MVC ViewImports                               viewimports          [C#]        Web/ASP.NET
MVC ViewStart                                 viewstart            [C#]        Web/ASP.NET
Blazor Server App                             blazorserver         [C#]        Web/Blazor
Blazor WebAssembly App                        blazorwasm           [C#]        Web/Blazor/WebAssembly
ASP.NET Core Empty                            web                  [C#],F#     Web/Empty
ASP.NET Core Web App (Model-View-Controller)  mvc                  [C#],F#     Web/MVC
ASP.NET Core Web App                          webapp               [C#]        Web/MVC/Razor Pages   
ASP.NET Core with Angular                     angular              [C#]        Web/MVC/SPA
ASP.NET Core with React.js                    react                [C#]        Web/MVC/SPA
ASP.NET Core with React.js and Redux          reactredux           [C#]        Web/MVC/SPA
Razor Class Library                           razorclasslib        [C#]        Web/Razor/Library     
ASP.NET Core Web API                          webapi               [C#],F#     Web/WebAPI
ASP.NET Core gRPC Service                     grpc                 [C#]        Web/gRPC
dotnet gitignore file                         gitignore                        Config
global.json file                              globaljson                       Config
NuGet Config                                  nugetconfig                      Config
Dotnet local tool manifest file               tool-manifest                    Config
Web Config                                    webconfig                        Config
Solution File                                 sln                              Solution
Protocol Buffer File                          proto                            Web/gRPC

Examples:
    dotnet new mvc --auth Individual
    dotnet new web 
    dotnet new --help
    dotnet new nunit --help
```

Создадим консольное приложение командой: ``` dotnet new console ```

Но как теперь добавить в наш проект NuGet пакеты также из терминала?! Install-Package не работает :(

Тут нам поможет команда: ``` dotnet add package <ИМЯ_ПАКЕТА> ```

Например, добавим в наш проект пакеты  **BenchmarkDotNet** и **BenchmarkDotNet.Annotations**.
Для этого в терминале выполним следующие команды:
``` dotnet add package BenchmarkDotNet ```
``` dotnet add package BenchmarkDotNet.Annotations ```

Вуаля! Требующиеся NuGet пакеты добавлены в наш проект:

``` xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net5.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="BenchmarkDotNet" Version="0.13.1" />
    <PackageReference Include="BenchmarkDotNet.Annotations" Version="0.13.1" />
  </ItemGroup>

</Project>
```
