---
layout: post
title: The Performance Penalty of the if/let Chain
comments: True
---

As a follow up from my previous post, I thought I'd get a little more specific and scientific on the subscript performance boost. I am using a Swift Playground to investigate - feel free to play along at home.

The scenario most relevant to this performance boost is using data loaded from a plist file in your app. So we start with a basic NSDictionary. Why an NSDictionary? Because loading a plist from the bundle requires you to use the NS classes in the Foundation framework. Unfortunately there's no native Swift Dictionary plist loader.

```swift
var nsDict = NSMutableDictionary()

let metadata1 = ["Director": "John McTiernan", "Tag Line": "40 Stories of Sheer Adventure!"]
let metadata2 = ["Director": "Renny Harlin",   "Tag Line": "Die Harder."]
let metadata3 = ["Director": "John McTiernan", "Tag Line": "Fun with a vengeance!"]
let metadata4 = ["Director": "Len Wiseman",    "Tag Line": ""]

let movies = ["Die Hard": metadata1,
    "Die Hard 2": metadata2,
    "Die Hard 3": metadata3,
    "Die Hard 4": metadata4]

nsDict["Movies"] = movies
```

What we're going to be testing here is getting the metadata for a movie out of this dictionary. Let's set up a functional function as a scaffold for our testing. We just inject whatever method of fetching we want to test into it.

```swift
typealias Metadata = String -> [String: String]?

func getMetadataManyTimes(repeats: Int, getFunction: Metadata) {
    for var x=0; x<repeats; x++ {
        if let metadata = getFunction("Die Hard 2") {
            metadata
        }
    }
}
```

And some vars for later use.

```swift
let repeats = 5000
var timeTaken: NSTimeInterval
var percentFaster: Double
```

First let's measure the if/let version of getting the metadata as a baseline.

```swift
//
// NSDictionary using sequence of if/let
//

var startDate = NSDate()

getMetadataManyTimes(repeats) { title in
    if let movies = nsDict["Movies"] as? [String: AnyObject],
        metadata = movies[title] as? [String: String] {
            return metadata
    }
    return nil
}

let baseline = NSDate().timeIntervalSinceDate(startDate)
```

Now our first comparison, optimising by using chained subscripts.

```swift
//
// NSDictionary using chained subscripts
//

startDate = NSDate()

getMetadataManyTimes(repeats) { title in
    return nsDict["Movies"]?[title] as? [String: String]
}

timeTaken = NSDate().timeIntervalSinceDate(startDate)
percentFaster = ((baseline / timeTaken) - 1 ) * 100
```

```percentFaster``` is always >10% on my machine, usually around 20%. That's not bad! Not quite the 156ms down to 36ms from my previous post, but a function called sporadically during UIView loading has much more complexity and flow on effects around it.

Let's try using a strongly typed Swift version. No bridging, no casting.

```swift
//
// Swift Dictionary using chained subscripts
//

var swiftDict = [String: [String: [String: String]]]()
swiftDict["Movies"] = movies

startDate = NSDate()

getMetadataManyTimes(repeats) { title in
    return swiftDict["Movies"]?[title]
}

timeTaken = NSDate().timeIntervalSinceDate(startDate)
percentFaster = ((baseline / timeTaken) - 1 ) * 100
```

```percentFaster``` is always very close to the NSDictionary chained subscripts method - sometimes slightly faster, sometimes slightly slower. So performance wise you could go either way. But for code cleanliness this is great.

Just for kicks, let's try the NS valueForKeyPath:

```swift
//
// NSDictionary using valueForKeyPath
//

startDate = NSDate()

getMetadataManyTimes(repeats) { title in
    nsDict.valueForKeyPath("Movies" + title) as? [String: String]
}

timeTaken = NSDate().timeIntervalSinceDate(startDate)
percentFaster = ((baseline / timeTaken) - 1 ) * 100
```

```percentFaster``` hovers around 10%, but always slower than chained subscripts and straight Swift methods. Still, interesting to know that the NS valueForKeyPath is actually faster than doing a Swift if/let chain.

#### Concluding Thoughts

If you have a plist file that you know the structure for, do yourself a favour and map it into a strongly typed Swift dictionary after loading from the bundle. Your code and your users will thank you for it. Otherwise, try and avoid the if/let chain and go for the optional subscript chain if you can.

Next post I will be investigating other scenarios where if/let can be substituted with other options, and measuring the performance benefit or cost in doing so. I have a hunch that if/let is a little taxing and should be avoided where possible. And no this is not a case for forced unwrapping!! Forced unwrapping is the anti-Swift.
