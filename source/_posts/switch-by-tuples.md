---
title: Сопоставление с образцом на основе кортежей
date: 2022-04-22 18:57:13
tags: [C#, switch]
---

Ранее мною уже был рассмотрен один из механизмов сопоставления с образцом (паттерн матчинга) в статье [Сопоставление с образцом на основе значений свойств объекта](https://ostart.github.io/2022/03/17/switch-by-property-values/). В данной статье рассмотрим сопоставление с образцом на основе кортежей, что делает код намного короче.

Рассмотрим класс комнаты и перечисление статуса члена клуба такого вида:

``` csharp
public class Room
{
    public int Number { get; set; }
    public string Type { get; set; }
    public string Size { get; set; }

    public void Deconstruct(out string type, out string size)
    {
        type = type;
        size = Size;
    }
}

public enum Membership
{
    Bronze,
    Silver,
    Gold
}
```

Метод ```GetRooms``` возвращает список комнат вида:

``` csharp
public static List<Room> GetRooms()
{
    return new List<Room>
    {
        new Room
        {
            Number = 3,
            Size = "Large",
            Type = "Deluxe"
        },
        new Room
        {
            Number = 2,
            Size = "Large",
            Type = "Regular"
        },
        new Room
        {
            Number = 1,
            Size = "Standard",
            Type = "Regular"
        }
    };
}
```

Начиная с C# 8.0 появилась возможность сопоставлять с образцом в выражении ```switch```, используя деконструктор класса (см. ранее конструкцию ```Deconstruct``` в определении класса ```Room```):

``` csharp
const int RoomNotAvailable = -1;

public static int AssignRoom(Membership membership)
{
    foreach(var room in GetRooms())
    {
        Membership clientCategory = room switch
        {
            ("Deluxe", "Large") => Membership.Gold,
            ("Regular", "Large") => Membership.Silver,
            ("Regular", "Standard") => Membership.Bronze,
            _ => Membership.Bronze
        };

        if (clientCategory == membership)
            return room.Number;
    }

    return RoomNotAvailable;
}
```

Код ветки ```_ => Membership.Bronze``` аналогичен случаю ```default:``` в стандартном ```switch```.

Тогда основной метод программы может использовать вышеприведённый код таким образом:

``` csharp
static voir Main()
{
    Console.Write("Choose status: 0 for Bronze, 1 for Silver, 2 for Gold");
    string clientStatus = Console.ReadLine();

    Enum.TryParse(clientStatus, out Membership membership);

    int roomNumber = AssignRoom(membership);

    if (roomNumber == RoomNotAvailable)
        Console.WriteLine("Room not available");
    else
        Console.WriteLine($"Room number is {roomNumber}");
}
```

Видно на сколько короче стала конструкция ```switch``` благодаря использованию кортежей. Компактный синтаксис кортежей делает их идеальным инструментом для сопоставления с образцом (паттерн матчинга). В остальном код и результаты идентичны приведённому в статье [Сопоставление с образцом на основе значений свойств объекта](https://ostart.github.io/2022/03/17/switch-by-property-values/).