---
title: Сопоставление с образцом на основе значений свойств объекта
date: 2022-03-17 10:26:48
tags: [C#, switch]
---

Рассмотрим класс комнаты и перечисление статуса члена клуба такого вида:

``` csharp
public class Room
{
    public int Number { get; set; }
    public string Type { get; set; }
    public string Size { get; set; }
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

Начиная с C# 8.0 появилась возможность сопоставлять с образцом в выражении ```switch```, используя несколько значений свойств объекта класса:

``` csharp
const int RoomNotAvailable = -1;

public static int AssignRoom(Membership membership)
{
    foreach(var room in GetRooms())
    {
        Membership clientCategory = room switch
        {
            { Size: "Large", Type: "Deluxe" } => Membership.Gold,
            { Size: "Large", Type: "Regular" } => Membership.Silver,
            { Size: "Standard", Type: "Regular" } => Membership.Bronze,
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

Основная магия сопоставления с образцом с использованием значений свойств объектов происходит в методе ```AssignRoom```. Раньше можно было делать ```switch``` только на основе единственного значения, но начиная с С# 8.0 паттерн матчинг можно делать на основе значений нескольких свойств экзмепляров классов. Такой подход намного более читабельный и улучшает ясность кода.