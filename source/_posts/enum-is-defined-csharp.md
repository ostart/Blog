---
title: Enum.IsDefined и с чем его едят
date: 2021-09-18 09:20:27
tags: [C#, Enum, BenchmarkDotNet]
---

Работа с перечислениями (Enum) в C# имеет очень большие издержки. Это странно, т.к. перечисления целочисленны по своей природе, но из-за требований безопасности типов даже простые операции обходятся дорого. Подробно этот аспект освещён в книге Бена Уотсона "Высокопроизводительный код на платформе .NET" (2-е издание) в главе 6 "Использование среды .NET Framework" раздел "Удивительно высокие издержки использования перечислений". Там рассмотрено устройство функции Enum.HasFlag, а анализ функции Enum.IsDefined оставлен в виде домашнего задания. Собственно займёмся этой функцией, учтя основную рекомендацию Бена: **"если окажется, что проверять наличие флага приходится часто, реализуйте проверку самостоятельно"**. ILSpy показывает следующее устройство функции IsDefined:

``` csharp
// System.Enum
public static bool IsDefined(Type enumType, object value)
{
	if ((object)enumType == null)
	{
		throw new ArgumentNullException("enumType");
	}
	return enumType.IsEnumDefined(value);
}

// System.Type
public virtual bool IsEnumDefined(object value)
{
	if (value == null)
	{
		throw new ArgumentNullException("value");
	}
	if (!IsEnum)
	{
		throw new ArgumentException(SR.Arg_MustBeEnum, "value");
	}
	Type type = value.GetType();
	if (type.IsEnum)
	{
		if (!type.IsEquivalentTo(this))
		{
			throw new ArgumentException(SR.Format(SR.Arg_EnumAndObjectMustBeSameType, type, this));
		}
		type = type.GetEnumUnderlyingType();
	}
	if (type == typeof(string))
	{
		string[] enumNames = GetEnumNames();
		object[] array = enumNames;
		if (Array.IndexOf(array, value) >= 0)
		{
			return true;
		}
		return false;
	}
	if (IsIntegerType(type))
	{
		Type enumUnderlyingType = GetEnumUnderlyingType();
		if (enumUnderlyingType.GetTypeCodeImpl() != type.GetTypeCodeImpl())
		{
			throw new ArgumentException(SR.Format(SR.Arg_EnumUnderlyingTypeAndObjectMustBeSameType, type, enumUnderlyingType));
		}
		Array enumRawConstantValues = GetEnumRawConstantValues();
		return BinarySearch(enumRawConstantValues, value) >= 0;
	}
	throw new InvalidOperationException(SR.InvalidOperation_UnknownEnumType);
}
```
Выглядит такой код чрезмерно сложно. Попробуем реализовать метод IsDefined самостоятельно. Для этого преобразуем перечисление в словарь **equivalentEnumDictionary** внутри класса **MyEnum**. Тогда эквивалентом метода **IsDefined** будет метод **MyIsDefined** с таким же набором аргументов:

``` csharp
using System;
using System.Linq;
using System.Collections.Generic;

namespace EnumSharp
{
    class MyEnum
    {
        private static Dictionary<int, string> equivalentEnumDictionary;

        public static bool MyIsDefined(Type enumType, object value)
        {
            if (equivalentEnumDictionary == null) 
            {
                equivalentEnumDictionary = Enum.GetValues(enumType)
                                               .Cast<object>()
                                               .ToDictionary(k => (int)k, v => ((Enum)v).ToString());
            }

            if (value is int && equivalentEnumDictionary.ContainsKey((int)value))
                return true;

            if (value is string && equivalentEnumDictionary.ContainsValue((string)value))
                return true;

            if (value.GetType() == enumType && equivalentEnumDictionary.ContainsValue(value.ToString()))
                return true;
            
            return false; 
        }
    }
}
```

Преобразование перечисления в словарь произойдёт единожды при первом обращении к методу **MyIsDefined**.

Измерим, насколько самостоятельная проверка **MyIsDefined** быстрее встроенной **IsDefined**. Для этого воспользуемся библиотекой **BenchmarkDotNet**.

``` csharp
using System;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

namespace EnumSharp
{
    [Flags] 
    public enum PetType
    {
        None = 0, Dog = 1, Cat = 2, Rodent = 4, Bird = 8, Reptile = 16, Other = 32
    };

    public class EnumsBenchmark
    {
        [Benchmark]
        public void MethodEnumIsDefined()
        {
            bool result;

            result = Enum.IsDefined(typeof(PetType), 1); // true
            result = Enum.IsDefined(typeof(PetType), 64); // false
            result = Enum.IsDefined(typeof(PetType), "Rodent"); // true
            result = Enum.IsDefined(typeof(PetType), PetType.Dog); // true
            result = Enum.IsDefined(typeof(PetType), PetType.Dog | PetType.Cat); // false
            result = Enum.IsDefined(typeof(PetType), "None"); // true
            result = Enum.IsDefined(typeof(PetType), "NONE"); // false
        }

        [Benchmark]
        public void MethodMyIsDefined()
        {
            bool result;

            result = MyEnum.MyIsDefined(typeof(PetType), 1); // true
            result = MyEnum.MyIsDefined(typeof(PetType), 64); // false
            result = MyEnum.MyIsDefined(typeof(PetType), "Rodent"); // true
            result = MyEnum.MyIsDefined(typeof(PetType), PetType.Dog); // true
            result = MyEnum.MyIsDefined(typeof(PetType), PetType.Dog | PetType.Cat); // false
            result = MyEnum.MyIsDefined(typeof(PetType), "None"); // true
            result = MyEnum.MyIsDefined(typeof(PetType), "NONE"); // false
        }

        class Program
        {
            static void Main(string[] args)
            {
                var summary = BenchmarkRunner.Run<EnumsBenchmark>();
            }
        }
    }
}
```

И вот результаты:

```
|              Method |     Mean |    Error |   StdDev |
|-------------------- |---------:|---------:|---------:|
| MethodEnumIsDefined | 845.3 ns | 15.14 ns | 13.42 ns |
|   MethodMyIsDefined | 350.2 ns |  6.95 ns |  6.83 ns |

```

Получается, что самостоятельная проверка **IsDefined** более чем в два раза быстрее даже при такой прямолинейной реализации. Стоит согласиться с Беном Уотсоном, что если проверять **IsDefined** приходится часто, то реализация самостоятельной проверки даст существенный профит. При обычном сценарии достаточно пользоваться встроенной **IsDefined**.