---
title: Как изменить значение в JSON прочитав значение из другого узла с помощью SQL запроса
date: 2024-02-01 11:46:35
tags: [MicrosoftSQLServer, PostgreSQL]
---

В предыдущей статье [Как добавить узел в JSON с помощью SQL запроса](https://ostart.github.io/2023/08/04/insert-node-to-json-by-sql/) было рассмотрено, как воспользовавшись средствами самого SQL вставить новый узел в сериализованный JSON объект, хранящийся в ячейке БД. В данной статье рассмотрим, как изменить значение по ключу в JSON объекте, предварительно считав его из другого поля JSON объекта.

Как это можно сделать в ```PostgreSQL```:
``` sql
START TRANSACTION;

DO $$
DECLARE json_data JSONB;
DECLARE len INT;
DECLARE path text[];
DECLARE base text;
DECLARE filter text;
BEGIN
  -- Читаем существующие JSON данные из поля Configuration
  SELECT "Configuration" INTO json_data FROM "IdentityIntegration" WHERE "Priority" = 1;

  -- Вытаскиваем длину узла Servers, который представлет из себя массив json объектов
  SELECT jsonb_array_length(json_data->'Servers') into len;

  -- Для каждого элемента массива Servers
  FOR i IN 0..len-1 LOOP
	    -- формируем путь к ключу
        path := ARRAY['Servers', i::text, 'IsReferral'];
	    -- устанавливаем значение json_data по ключу path в false
        json_data = jsonb_set(json_data, path, 'false'::jsonb, true);

	    -- считываем значения из json объекта json_data в переменные base и filter
        SELECT json_data->'Servers'->i->'Sources'->0->'Base' into base;
        SELECT json_data->'Servers'->i->'Sources'->0->'Filter' into filter;

        -- формируем путь к ключу
        path := ARRAY['Servers', i::text, 'GroupSource'];

        --RAISE NOTICE 'Iteration: %', ('{"Base": ' || base || ', "Filter": ' || filter || '}'); -- печатаем в консоль если требуется

	    -- устанавливаем значение json_data по ключу path в сложный объект, получающийся конкатенацией переменных base и filter
        json_data = jsonb_set(json_data, path, ('{"Base": ' || base || ', "Filter": ' || filter || '}')::jsonb, true);
  END LOOP;

  -- Обновляем поле Configuration изменённым значением JSON данных
  UPDATE "IdentityIntegration" SET "Configuration" = json_data WHERE "Priority" = 1;
END $$;

COMMIT;
```

Последний параметр со значения **true** в функции ```jsonb_set``` именуется **create_if_missing** и является опциональным.

Как это можно сделать в ```MSSQL```:
``` sql
BEGIN TRANSACTION;
GO

DECLARE @json_data NVARCHAR(MAX);
DECLARE @len INT;
DECLARE @path NVARCHAR(MAX);
DECLARE @base NVARCHAR(MAX);
DECLARE @filter NVARCHAR(MAX);

SET @json_data = (SELECT [Configuration] FROM [IdentityIntegration] WHERE [Priority] = 1);

SET @len = (SELECT COUNT(*) FROM OPENJSON(JSON_QUERY(@json_data, '$.Servers')));
-- PRINT @len; -- печатаем в консоль если требуется

DECLARE @i INT;
SET @i = 0;

WHILE @i < @len
BEGIN
    SET @path = CONCAT('$.Servers[', @i, '].IsReferral');
    SET @json_data = JSON_MODIFY(@json_data, @path, 'false');

    SET @base = JSON_VALUE(@json_data, CONCAT('$.Servers[', @i, '].Sources[0].Base'));
    SET @filter = JSON_VALUE(@json_data, CONCAT('$.Servers[', @i, '].Sources[0].Filter'));

    SET @path = CONCAT('$.Servers[', @i, '].GroupSource');
    -- применение функции JSON_QUERY(...) к содержимому изменяемого узла удаляет избыточно экранированные кавычки
    SET @json_data = JSON_MODIFY(@json_data, @path, JSON_QUERY(CONCAT('{"Base": "', @base, '", "Filter": "', @filter, '"}')));

    SET @i = @i + 1;
END;

UPDATE [IdentityIntegration] SET [Configuration] = @json_data WHERE [Priority] = 1;

COMMIT;
GO
```

Обращаю внимание на необходимость применения функции ```JSON_QUERY(...)``` к содержимому изменяемого узла, иначе вставляемые кавычки будут избыточно экранированы.
