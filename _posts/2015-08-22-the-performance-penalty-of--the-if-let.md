---
layout: post
title: The Performance Penalty of the if/let Chain
comments: True
---

As a follow up from my previous post, I thought I'd get a little more specific and scientific on the subscript performance boost. I am using a Swift Playground to investigate - feel free to play along at home.

The scenario investigated here is where you are accessing data loaded from a plist file in your app. When we load a plist, we unfortunately have to do it via an NSDictionary, and then bridge to the Swift Dictionary class and methods.

```swift
let metadata1 = ["Director": "John McTiernan",
                 "Tag Line": "40 Stories of Sheer Adventure!"]
let metadata2 = ["Director": "Renny Harlin",
                 "Tag Line": "Die Harder."]
let metadata3 = ["Director": "John McTiernan",
                 "Tag Line": "Fun with a vengeance!"]
let metadata4 = ["Director": "Len Wiseman",
                 "Tag Line": ""]

let movies = ["Die Hard":   metadata1,
              "Die Hard 2": metadata2,
              "Die Hard 3": metadata3,
              "Die Hard 4": metadata4]

let nsDict = NSDictionary(objects: [movies], forKeys: ["Movies"])
```

What we're going to be testing here is accessing information nested a couple of levels deep. Let's set up a benchmark function to run our execute our different methods and return the total time taken.

```swift
typealias NestedDict = () -> [String: String]?

func benchmarkGetNestedDict(repeats: Int, getDict: NestedDict) -> NSTimeInterval {
    let startDate = NSDate()
    for var x=0; x<repeats; x++ {
        getDict()
    }
    return NSDate().timeIntervalSinceDate(startDate)
}
```

Now let's set everything up for the comparisons: some constants and variables for the benchmarks, and the if/let method of fetching as a baseline to compare against.

```swift
let movieTitle = "Die Hard 2"
let repeats = 5000
var comparison: NSTimeInterval
var percentFaster: Double

let baseline = benchmarkGetNestedDict(repeats) {
    if let movies = nsDict["Movies"] as? [String: AnyObject],
        metadata = movies[movieTitle] as? [String: String] {
            return metadata
    }
    return nil
}
```

Our first comparison: chained optional subscripts.

```swift
comparison = benchmarkGetNestedDict(repeats) {
    return nsDict["Movies"]?[movieTitle] as? [String: String]
}
percentFaster = ((baseline / comparison) - 1 ) * 100   // ~22%
```

Not bad! Still not quite the 156ms down to 36ms from my previous post, but in that instance we were testing a full app and not the specific code in a tight loop, so there was much more complexity surrounding the function in question.

Ok, Let's try using a strongly typed Swift version. No bridging, no casting.

```swift
let swiftDict = ["Movies": movies]
comparison = benchmarkGetNestedDict(repeats) {
    return swiftDict["Movies"]?[movieTitle]
}
percentFaster = ((baseline / comparison) - 1 ) * 100   // ~23%
```

Not bad again. Slightly faster this way, but then the bridged NSDictionary is essentially just a Swift dictionary, so all we've really done here is taken off the optional cast to ```[String: String]```. Still, this gets a big tick from me - the cleaner the code the better.

Now just for kicks, let's try using NSDictionary's valueForKeyPath:

```swift
comparison = benchmarkGetNestedDict(repeats) {
    nsDict.valueForKeyPath("Movies." + movieTitle) as? [String: String]
}
percentFaster = ((baseline / comparison) - 1 ) * 100   // ~16%
```

Slower than optional subscripts and casting, but still faster than the if/let chain! This surprised me, I would have thought ```valueForKeyPath``` would have had more performance penalty being a legacy Foundation method.

#### Concluding Thoughts

For fastest and cleanest code, get that NSDictionary into a strongly typed nested Swift Dictionary and access it with optional subscripts. In cases where you might not know the structure of your data so precisely, go for the optional subscripts and casting. In any case, avoid the if/let!

You may be wondering about 1 ommision from these benchmarks - force casting. I personally consider force casting the anti-Swift, and only use it where I absolutely have to (which is basically never), so I didn't feel the need to investigate.

Next post I will be taking a look into other scenarios where if/let can be substituted for optional chains, and measuring the performance benefit or cost in doing so. I have a hunch that if/let is a little taxing and if there's a safe way of using optional chains then this is the more optimised route. We shall see....
