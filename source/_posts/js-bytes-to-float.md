---
title: Преобразование массива байтов во float в JavaScript
date: 2021-09-07 18:08:00
tags: [JavaScript, JS, ClearScript]
---

Если вам требует в JavaScript преобразовать массив из 4-х байтов в число с плавающей точкой единичной точности (float), то можно воспользоваться следующей функцией:

``` javascript
function bytesToFloat(bytes) {
    const buffer = new ArrayBuffer(4);
    const view = new DataView(buffer, 0 , 4); // для ClearScript нужен конструктор со всеми параметрами
    for (let i = 0; i < bytes.length; i += 1) {
        view.setUint8(i, bytes[i]);
    }
    return view.getFloat32(0);
}
```

Обращу внимание на конструкцию вида:

``` javascript
const view = new DataView(buffer, 0 , 4);
```

В теории, при инициализации объекта класса DataView через конструктор должен создаваться *view* того же размера, что и *buffer*. Но на практике применения конструктора DataView в ClearScript (см. [предыдущий пост с объяснениями что такое ClearScript](https://ostart.github.io/2021/09/07/clearscript/)) выяснилось, что требуется использовать перегрузку конструктора DataView с указанием всех параметров, иначе ClearScript порождает ошибку выполнения с сообщением о том, что view создаётся меньшего размера, чем требуется для buffer.