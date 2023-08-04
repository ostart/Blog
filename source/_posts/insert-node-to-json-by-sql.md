---
title: Как добавить узел в JSON с помощью SQL запроса
date: 2023-08-04 13:21:13
tags: [MicrosoftSQLServer, PostgreSQL]
---

Рассмотрим ситуацию, когда в Базе Данных (БД) имеется сохранённый в виде строки сериализованный JSON объект, а нам требуется добавить в него новый узел. Конечно, можно взять язык программирования, считать значение из БД, изменить его и перезаписать в БД. Но хочется воспользоваться средствами самого SQL для подобной операции.

Как это можно сделать в ```MSSQL```:
``` sql
DECLARE @json NVARCHAR(2000);

-- Читаем существующие JSON данные из поля Configuration
SET @json = (SELECT [Configuration] FROM [dbo].[Integration] WHERE [Priority] = 1);

-- Изменим JSON данные добавив в них новый узел Cache
SET @json = JSON_MODIFY(@json, '$.Cache', JSON_QUERY('{"Interval":1,"IntervalDimension":"day"}'));

-- Обновим поле Configuration изменённым значением JSON данных
UPDATE [dbo].[Integration]
SET [Configuration] = @json
WHERE [Priority] = 1;
```

Обращаю внимание на необходимость применения функции ```JSON_QUERY(...)``` к содержимому добавляемого узла, иначе вставляемые кавычки будут избыточно экранированы.

Как это можно сделать в ```PostgreSQL```:
``` sql
DO $$
DECLARE json_data JSONB;
BEGIN
  SELECT "Configuration" INTO json_data FROM "Integration" WHERE "Priority" = 1;
  json_data = jsonb_set(json_data, '{Cache}', '{"Interval": 1, "IntervalDimension": "day"}', true);
  UPDATE "Integration" SET "Configuration" = json_data WHERE "Priority" = 1;
END $$;
```

Последний параметр со значения **true** в функции ```jsonb_set``` именуется **create_if_missing** и является опциональным.

В случае удаления ненужного узла из JSON объекта требуется установить значение узла в ```NULL```, как показано ниже для  ```MSSQL```:
``` sql
DECLARE @json NVARCHAR(2000);

SET @json = (SELECT [Configuration] FROM [dbo].[Integration] WHERE [Priority] = 1);

SET @json = JSON_MODIFY(@json, '$.Cache', NULL);

UPDATE [dbo].[Integration]
SET [Configuration] = @json
WHERE [Priority] = 1;
```
