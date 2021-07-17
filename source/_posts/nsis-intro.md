---
title: О NSIS
date: 2021-07-17 13:41:09
tags: [NSIS, nsis]
---

Если вам однажды потребуется изготовить установщик под Windows, то рекомендую [NSIS (Nullsoft Scriptable Install System)](https://nsis.sourceforge.io/). NSIS - это система с открытым исходным кодом, достаточно компактная и гибкая. У неё есть свои минусы (низкоуровневый синтаксис, включающий работу с регистрами, макросы и т.п.), но плюсы, как правило, их перевешивают. 

### К основным плюсам я бы отнёс: 
* сверх компактный размер (оверхэд всего 34 KB)
* большое количество плагинов, позволяющих реализовать почти все задумки
* безоплатность
* стабильность работы
* совместимость со всеми версиями: Windows 95, Windows 98, Windows ME, Windows NT, Windows 2000, Windows XP, Windows Server 2003, Windows Vista, Windows Sever 2008, Windows 7, Windows Server 2008R2, Windows 8, Windows Server 2012, Windows 8.1, Windows Server 2012R2, Windows Server 2016 и Windows 10

Плагины включают в себя взаимодействие с БД, реестром, файлами, Windows службами, IIS, командной строкой, Powershell, Win32 API вызовы, создание ярлыков, деинсталляторов и многое многое другое.

Взаимодейтсвие с NSIS основано на скриптах, которые транслируются в Windows установщик. NSIS до сих пор поддерживается (последний релиз NSIS 3.06.1 от 31 июля 2020), имеет большое сообщество и массу туториалов. 

Пример самого простого NSIS скрипта:
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

Для работы со скриптами я рекомендую воспользоваться программой [Venis IX](https://nsis.sourceforge.io/Venis_IX). О работе с ней будет отдельный пост.

В конце хотел бы привести несколько типичных изображений работы установщика NSIS с которым каждый пользователь Windows хоть раз, да сталкивался:

{% asset_img nsis-intro1.png NSIS Intro %}
&nbsp;
{% asset_img nsis-intro2.png NSIS Intro %}
