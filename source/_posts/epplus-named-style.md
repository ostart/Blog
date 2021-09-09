---
title: Как эффективно выделять ячейки в файлах Excel с помощью EPPlus
date: 2021-09-09 14:08:53
tags: [C#, EPPlus, Excel]
---

При построении отчетов в различных системах используется Excel. Для его формирования из C# кода часто используется [EPPlus](https://epplussoftware.com/). EPPlus интуитивно понятен:

``` csharp
var file = new FileInfo(@"c:\temp\myWorkbook.xlsx");
using(var package = new ExcelPackage(file))
{
    var sheet = package.Workbook.Worksheets.Add("My Sheet");
    sheet.Cells["A1"].Value = "Hello World!";

    // Save to file
    package.Save();
}
```

EPPlus имеет много встроенных механизмов для получения одного и того же результата разными путями, но не все пути одинаково эффективны для больших и очень больших размеров рабочих листов. В частности, я столкнулся с проблемой обведения контуров ячеек и выделения жирным шрифтом документов Excel, содержащих свыше 10000 столбцов.

Используемый мной подход "в лоб" заключался в обводке контуров ячеек непосредственно в месте их заполнения данными с помощью функции DrawBorder:

``` csharp
// использование функции DrawBorder. ПОЛНЫЙ ПРОВАЛ в эффективности!!!
DrawBorder(worksheet.Cells[4, 6, 4, shift + dateTimeRanges.Count - 1], ExcelBorderStyle.Medium, BorderSide.TopBottom);
DrawBorder(worksheet.Cells[4, 6, 4, shift + dateTimeRanges.Count - 1], ExcelBorderStyle.Thin, BorderSide.LeftRight);
DrawBorder(worksheet.Cells[4, shift + dateTimeRanges.Count - 1], ExcelBorderStyle.Medium, BorderSide.Right);

// определение функции DrawBorder: НАИВНОЕ и НЕЭФФЕКТИВНОЕ
private static void DrawBorder(ExcelRange worksheetCell, ExcelBorderStyle borderStyle, BorderSide side)
{
    switch (side)
    {
        case BorderSide.EveryOne:
            worksheetCell.Style.Border.Top.Style = borderStyle;
            worksheetCell.Style.Border.Left.Style = borderStyle;
            worksheetCell.Style.Border.Right.Style = borderStyle;
            worksheetCell.Style.Border.Bottom.Style = borderStyle;
            break;
        case BorderSide.Top:
            worksheetCell.Style.Border.Top.Style = borderStyle;
            break;
        case BorderSide.Left:
            worksheetCell.Style.Border.Left.Style = borderStyle;
            break;
        case BorderSide.Bottom:
            worksheetCell.Style.Border.Bottom.Style = borderStyle;
            break;
        case BorderSide.Right:
            worksheetCell.Style.Border.Right.Style = borderStyle;
            break;
        case BorderSide.TopBottom:
            worksheetCell.Style.Border.Top.Style = borderStyle;
            worksheetCell.Style.Border.Bottom.Style = borderStyle;
            break;
        case BorderSide.LeftRight:
            worksheetCell.Style.Border.Left.Style = borderStyle;
            worksheetCell.Style.Border.Right.Style = borderStyle;
            break;
        default:
            throw new ArgumentOutOfRangeException(nameof(side), side, null);
    }   
}
```

Данный подход показал свою **полную неэффективность**. Отчеты строились по часу и более...

Правильный подход заключается в определении именованных стилей NamedStyle и применении этих стилей для конкретных ячеек:

``` csharp
// определение именованных стилей NamedStyle
private static void AddNamedStyles(ExcelPackage excel)
{
    var thinBorders = excel.Workbook.Styles.CreateNamedStyle("ThinBordersStyle");
    thinBorders.Style.Border.Top.Style = ExcelBorderStyle.Thin;
    thinBorders.Style.Border.Left.Style = ExcelBorderStyle.Thin;
    thinBorders.Style.Border.Right.Style = ExcelBorderStyle.Thin;
    thinBorders.Style.Border.Bottom.Style = ExcelBorderStyle.Thin;
    thinBorders.Style.HorizontalAlignment = ExcelHorizontalAlignment.Center;
    thinBorders.Style.VerticalAlignment = ExcelVerticalAlignment.Center;

    ...

    var totalThinBoldBorders = excel.Workbook.Styles.CreateNamedStyle("TotalThinBordersBoldStyle");
    totalThinBoldBorders.Style.Border.Top.Style = ExcelBorderStyle.Thin;
    totalThinBoldBorders.Style.Border.Left.Style = ExcelBorderStyle.Thin;
    totalThinBoldBorders.Style.Border.Right.Style = ExcelBorderStyle.Thin;
    totalThinBoldBorders.Style.Border.Bottom.Style = ExcelBorderStyle.Thin;
    totalThinBoldBorders.Style.HorizontalAlignment = ExcelHorizontalAlignment.Center;
    totalThinBoldBorders.Style.VerticalAlignment = ExcelVerticalAlignment.Center;
    totalThinBoldBorders.Style.Font.Bold = true;
    totalThinBoldBorders.Style.Font.Size = 12;
}

// использование именованных стилей NamedStyle
worksheet.Cells[startRow, 1, currentRow - 1, shift + dateTimeRanges.Count - 1].StyleName = "ThinBordersStyle";
worksheet.Cells[currentRow, 1, currentRow + 1, shift + dateTimeRanges.Count - 1].StyleName = "TotalThinBordersBoldStyle";
```

Использование именованных стилей NamedStyle привело к намного более эффективному построению отчетов Excel. Отчеты стали строиться за 7-10 минут, что почти в 10 раз быстрее первоначального подхода с DrawBorder. Как говорилось в рекламе из 90-х, "не все йогурты одинаково полезны..."
