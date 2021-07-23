---
title: Работа с NSIS скриптами в Venis_IX
date: 2021-07-23 11:34:07
tags: [Venis_IX, NSIS, nsis, script]
---

Для редактирования и компиляции NSIS скриптов есть замечательная программа [Venis_IX](https://nsis.sourceforge.io/Venis_IX).

{% asset_img venis1.jpg Venis Main Window %}

Для шаблонной генерации скрипта установщика в программе присутствует Venis Install Wizard:

{% asset_img venis2.jpg Venis Install Wizard %}

В основном окне можно создать / открыть скрипт (расширение .nsi), отредактировать его и откомпилировать. Результат компиляции отображается в консольном окне Compile Results. В случае успеха будет показан конечный размер установщика (Total size:) в байтах, как показано на изображении ниже:

{% asset_img venis3.jpg Venis Compile Results %}

В случае ошибки в консоли будет указана причина и строка на которой ошибка случилась.

{% asset_img venis4.jpg Venis Compile Error %}

К сожалению в программе нет привычного разработчикам дебаггера, поэтому отладка скрипта, как правило, осуществляется при его выполнении через всплывающие окна MessageBox или вывод в консоль с помощью DetailPrint.

Также в программе есть великолепная и подробная справка по языку, командам и макросам NSIS (NSIS User Manual):

{% asset_img venis5.jpg Venis NSIS Help %}
&nbsp;
{% asset_img venis6.jpg Venis NSIS User Manual %}

