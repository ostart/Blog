---
title: Как уменьшить размер файла логов в Microsoft SQL Server
date: 2022-12-21 12:53:39
tags: [MicrosoftSQLServer]
---
При создании Базы Данных (БД) в MS SQL Server для оптимизации логирования БД требуется установить *Recovery Mode* в значение **Simple**.

Для уже созданной БД с режимом *Recovery Mode* = **Full** требуется перейти в свойства БД и в разделе Options перевести соответствующую настройку из значения **Full**:

{% asset_img decrease-mssqlserver-logfile-size-1.png NSIS Intro %}

в значение **Simple**:

{% asset_img decrease-mssqlserver-logfile-size-2.png NSIS Intro %}

Затем необходимо очистить лишние записи действий в файле логов, т.к. модель восстановления данных *Recovery Mode* изменилась.

Для этого выберите требующуюся БД, затем кликните по ней Правой Кнопкой Мыши (ПКМ) и выберите *Tasks → Shrink → Files*:

{% asset_img decrease-mssqlserver-logfile-size-3.png NSIS Intro %}

После этого в появившемся окне формы **Shrink File** требуется установить *File type*: **Log** и выбрать **Release unused space**

{% asset_img decrease-mssqlserver-logfile-size-4.png NSIS Intro %}

Текущий размер занимаемого дискового пространства логами и свободное место для них можно оценить с помощью параметров *Currently allocated space* и *Available free space* этой же формы **Shrink File** (при повторном её открытии можно увидеть размеры после применения операции сжатия файла логов)
