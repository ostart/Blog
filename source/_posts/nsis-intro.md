---
title: О NSIS
date: 2021-07-17 13:41:09
tags: [NSIS, nsis]
---

Если вам однажды потребуется изготовить установщик для Windows, то обратите внимание на [NSIS (Nullsoft Scriptable Install System)](https://nsis.sourceforge.io/). NSIS - это профессиональная система с открытым исходным кодом, компактная и гибкая. У неё есть свои минусы (низкоуровневый синтаксис языка скриптов NSIS, включающий работу с регистрами, макросы и т.п.), но плюсы, как правило, их перевешивают. 

### К основным плюсам можно отнести: 
* сверх компактный размер (оверхэд самого NSIS всего 34 KB)
* совместимость с основными версиями ОС: Windows 95, Windows 98, Windows ME, Windows NT, Windows 2000, Windows XP, Windows Server 2003, Windows Vista, Windows Sever 2008, Windows 7, Windows Server 2008R2, Windows 8, Windows Server 2012, Windows 8.1, Windows Server 2012R2, Windows Server 2016 и Windows 10
* большое количество плагинов, позволяющих реализовать почти все задумки
* безоплатность
* стабильность
* многоязычность (в одном установщике до 60-ти языков)
* создание пользовательских диалоговых окон
* создание собственных плагинов (C, C++, Delphi)

Плагины включают в себя взаимодействие с БД, реестром, папками и файлами, переменными окружения, Windows службами, IIS, командной строкой, Powershell, Win32 API вызовы, перезагрузку системы, интернет соединения, создание ярлыков, деинсталляторов и многое многое другое.

Взаимодействие с NSIS основано на скриптах (на собственном языке программирования, базовый синтаксис которого описан в [посте]()), которые собираются в Windows установщик. NSIS до сих пор поддерживается (последний релиз NSIS 3.06.1 от 31 июля 2020), имеет большое сообщество и массу туториалов.

### Структура скрипта:
* Атрибуты установщика (Константы: WINDIR, SYSDIR, INSTDIR, PROGRAMFILES ... Атрибуты: BrandingText, Caption, Icon ...)
* Страницы (лицензия, компоненты, выбор директории, подтверждение деинсталляции ...)
* Секции (установка, деинсталляция ...)
* Функции (.onInit, un.onInit, .onGUIInit, .onInstSuccess, .onInstFailed, .onMouseOverSection ...)

Подробнее с синтаксисом NSIS можно ознакомиться тут: [Введение в синтаксис NSIS скриптов](https://ostart.github.io/2021/07/18/nsis-scripting/)

Пример простого NSIS скрипта:
``` nsis
!include "MUI2.nsh"
!include LogicLib.nsh

Name "SectionTest"
OutFile "SectionTest.exe"
ShowInstDetails show

!insertmacro MUI_PAGE_COMPONENTS
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_LANGUAGE "English"

Section "One" SecOne  
SectionEnd

Section "Two" SecTwo  
SectionEnd

Section "-OneTwo"
        ${If} ${SectionIsSelected} ${SecOne}
                DetailPrint "Section one"
                ${If} ${SectionIsSelected} ${SecTwo}
                        DetailPrint "Section two is selected"
                ${EndIf}
        ${EndIf}
        
        ${If} ${SectionIsSelected} ${SecTwo}
                DetailPrint "Section two"
        ${EndIf}
SectionEnd
```

Для работы со скриптами рекомендую пользоваться программой [Venis IX](https://nsis.sourceforge.io/Venis_IX). Работа с программой описана тут: [Работа с NSIS скриптами в Venis_IX](https://ostart.github.io/2021/07/23/venis-IX/).

В конце хотел бы привести несколько типичных изображений работы установщика NSIS с которым каждый пользователь Windows хоть раз, да сталкивался:

{% asset_img nsis-intro1.png NSIS Intro %}
&nbsp;
{% asset_img nsis-intro2.png NSIS Intro %}
