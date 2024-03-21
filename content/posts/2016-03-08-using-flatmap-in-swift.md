---
title: Using flatMap in Swift
date: 2016-03-08T18:21:00Z
tags: [swift, ios, dev]
---

I've come across the `flatMap` function several times. I've used `map` many times, but didn't understand why there was a need for `flatMap`? The function's description is quite concise and doesn't explain much (`transform` is the only closure parameter that this function accepts):

> Return an Array containing the concatenated results of mapping transform over self.

# "Flat" Array

The simplest and most obvious application of `flatMap` is to reduce the dimensionality of an array:

```swift
let array = [[0, 1, 2], [3, 4, 5]]
let flatArray = array.flatMap { $0 } // [0, 1, 2, 3, 4, 5]
```

Here we don't change the array's values. What if we need to get values ten times larger? The first implementation that comes to mind doesn't work:

```swift
let flatArray10 = array.flatMap { $0 * 10 }
```

The issue is that the argument in the closure taken by flatMap is not a number but an array, i.e., `[0, 1, 2]`, `[3, 4, 5]`. Therefore, you can get an array of numbers multiplied by 10 in this way:

```swift
let flatArray10 = array.flatMap { $0.map { $0 * 10 } }
```

# Optional

This function can work not only with arrays but also with optionals. For example, there is an array of numbers in string representation. This can be obtained, for example, in JSON, where numbers were placed in strings. You need to get an array of numbers so that you can work with them as with numbers, not strings. `Int` has `init?()` that takes a string and returns a number. The only problem is that it creates `Int?`, which is not very convenient. This is where `flatMap` can help. As an example, one of the strings is not a number, and its conversion result to `Int` will be `nil`:

```swift
let arrayOfStrings = ["0", "1", "a", "2"]
let arrayOfInts = arrayOfStrings.flatMap { Int($0) }
arrayOfInts // [0, 1, 2]
```

Another good example of such a technique can be creating an array `[UIImage]` from string image names `[String]`:

```swift
arrayOfImageNames.flatMap { UIImage(named: $0) }
```

You can also apply `flatMap` directly to an optional. For example, creating a URL from a string, which is of type `String?`. It can't be passed directly. And here `flatMap` comes to the rescue:

```swift
let urlString: String? = "http://apple.com"
let url: URL? = urlString.flatMap { URL(string: $0) }
```

## References:

- [NatashaTheRobot's Swift 2 flatMap](https://www.natashatherobot.com/swift-2-flatmap/)
- [SketchyTech Blog: Swift - What do map and flatMap really do?](http://sketchytech.blogspot.ru/2015/06/swift-what-do-map-and-flatmap-really-do.html)
- [Stack Overflow: How to use Swift flatMap to filter out optionals from an array](http://stackoverflow.com/questions/29870365/how-to-use-swift-flatmap-to-filter-out-optionals-from-an-array)
