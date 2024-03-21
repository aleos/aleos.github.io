---
title: Using flatMap in Swift
date: 2016-03-08T18:21:00Z
tags: [swift, ios, dev]
---

Мне несколько раз попадалась на глаза функция `flatMap`. Я много раз пользовался `map`, но не понимал, зачем нужен ещё какой-то `flatMap`? Описание функции весьма сжатое и мало что объясняет (`transform` — это единственный параметр-замыкание, который принимает эта функция):

> Return an Array containing the concatenated results of mapping transform over self.

# «Плоский» массив

Самое простое и наиболее очевидное применение `flatMap` заключается в уменьшении размерности массива:

```swift
let array = [[0, 1, 2], [3, 4, 5]]
let flatArray = array.flatMap { $0 } // [0, 1, 2, 3, 4, 5]
```

Здесь мы никак не меняем значения массива. Что, если нужно получить значения в 10 раз больше? Первая реализация, приходящая в голову, не работает:

```swift
let flatArray10 = array.flatMap { $0 * 10 }
```

Всё дело в том, что аргументом в замыкании, принимаемом flatMap, является в данном случае не число, а массив, т.е. `[0, 1, 2]`, `[3, 4, 5]`. Поэтому массив из чисел, умноженных на 10 можно получить таким образом:

```swift
let flatArray10 = array.flatMap { $0.map { $0 * 10 } }
```

# Optional

Эта функция умеет работать помимо массивов ещё и с optional'ами. Например, есть массив чисел, в строковом представлении. Такое можно получить к примеру в JSON, куда числа были положены в строках. Нужно получить массив чисел, чтобы с ними можно было работать как с числами, а не строками. У `Int` есть `init?()`, принимающий строку и возвращает число. Проблема только в том, что он создаёт `Int?`, что не очень удобно. В этом может как раз помочь `flatMap`. В качестве примера одна из строк не является числом и результатом её конвертации в `Int` будет `nil`:

```swift
let arrayOfStrings = ["0", "1", "a", "2"]
let arrayOfInts = arrayOfStrings.flatMap { Int($0) }
arrayOfInts // [0, 1, 2]
```

В качестве ещё одного хорошего примера такого приёма можно привести создание массива `[UIImage]` из строковых названий изображений `[String]`:

```swift
arrayOfImageNames.flatMap { UIImage(named: $0) }
```

На сам optional тоже можно применить `flatMap`. Например, создание URL из строки, которая имеет тип `String?`. Напрямую её передать нельзя. И тут на помощь приходит `flatMap`:

```swift
let urlString: String? = "http://apple.com"
let url: URL? = urlString.flatMap { URL(string: $0) }
```

## Ссылки:

- [NatashaTheRobot's Swift 2 flatMap](https://www.natashatherobot.com/swift-2-flatmap/)
- [SketchyTech Blog: Swift - What do map and flatMap really do?](http://sketchytech.blogspot.ru/2015/06/swift-what-do-map-and-flatmap-really-do.html)
- [Stack Overflow: How to use Swift flatMap to filter out optionals from an array](http://stackoverflow.com/questions/29870365/how-to-use-swift-flatmap-to-filter-out-optionals-from-an-array)
