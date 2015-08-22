---
layout: post
title: The performance penalty of the if/let chain
comments: True
---

As a follow up from my previous post, I thought I'd get a little more specific and scientific on the subscript performance boost. I am using a Swift Playground to investigate - feel free to play along at home.

The scenario investigated here is accessing data loaded from a plist file. When we load a plist we unfortunately have to do it via an NSDictionary and then bridge to the Swift Dictionary. So to replicate this in a Playground, lets build an NSDictionary of Die Hard movies and their respective directors and poster taglines. Yipee Kayee.

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

Let's set up a benchmark function to execute our different methods and return the total time taken.

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

Our first comparison: chained optional subscripts with an optional cast.

```swift
comparison = benchmarkGetNestedDict(repeats) {
    let metadata = nsDict["Movies"]?[movieTitle] as? [String: String]
    return metadata
}
percentFaster = ((baseline / comparison) - 1 ) * 100   // ~22%
```

22%, not bad! Still not quite the 156ms down to 36ms from my previous post, but in that instance we were testing a full app and not the specific code in a tight loop, so there was much more complexity surrounding the function in question.

Ok, let's try using a strongly typed Swift version. No bridging, no casting.

```swift
let swiftDict = ["Movies": movies]
swiftDict.dynamicType   // [String: [String: [String: String]]]
comparison = benchmarkGetNestedDict(repeats) {
    let metadata = swiftDict["Movies"]?[movieTitle]
    return metadata
}
percentFaster = ((baseline / comparison) - 1 ) * 100   // ~23%
```

Not bad again. Slightly faster this way, but then the bridged NSDictionary is essentially just a Swift dictionary, so all we've really done here is taken off the optional cast to ```[String: String]```. Still, this gets a big tick from me - the cleaner the code the better.

Now just for kicks, let's try using NSDictionary's valueForKeyPath:

```swift
comparison = benchmarkGetNestedDict(repeats) {
    let metadata = nsDict.valueForKeyPath("Movies." + movieTitle) as? [String: String]
    return metadata
}
percentFaster = ((baseline / comparison) - 1 ) * 100   // ~16%
```

Slower than optional subscripts and casting, but still faster than the if/let chain! This surprised me, I would have thought ```valueForKeyPath``` would have had more performance penalty being a legacy Foundation method.

#### Concluding Thoughts

For fastest and cleanest code, get that nested NSDictionary into a strongly typed nested Swift Dictionary and access it with optional subscripts. In cases where you might not know the structure of your data so precisely, access the NSDictionary with optional subscripts and casting. In any case, avoid the if/let!

You may be wondering about one omission from these benchmarks - force casting. I personally consider force casting the anti-Swift, and only use it where I absolutely have to (which is basically never), so I didn't feel the need to investigate.

Next post I will be taking a look into other scenarios where if/let can be substituted for optional chains, and measuring the performance benefit or cost in doing so. I have a hunch that if/let is a little taxing and if there's a safe way of using optional chains then this is the more optimised route. We shall see....
