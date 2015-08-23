---
layout: post
title: Swift's Dark Secret
comments: True
---

Since my last post I've been investigating different ways of accessing data in an NSDictionary in Swift and measuring the performance.

It turns out that it's not necessarily the if/let chain that causes a performance hit, but what was being done inside that if/let chain. Let me explain.

There was a flaw in the previous post. The results aren't entirely invalid, but I wasn't testing the scenario completely correctly. What I should have done was built a nested NSDictionary out of only NSDictionaries. Instead I mistakenly created an NSDictionary of Swift Dictionaries. I thought that since NSDictionary bridges to Dictionary, they're basically the same thing under the hood, just accessed different ways.... Well, they're not.

If we create a pure nested NSDictionary, like if we loaded a plist, we can access nested values like this:

```swift
if let movies = nsDict["Movies"] as? NSDictionary,
    metadata = movies["Die Hard 2"] as? NSDictionary,
    tagline = metadata["TagLine"] as? String {
        return tagline
}
```

It turns out that this is way faster than casting to the Swift equivalent every step.

```swift
if let movies = nsDict["Movies"] as? [String: AnyObject],
    metadata = movies["Die Hard 2"] as? [String: AnyObject],
    tagline = metadata["TagLine"] as? String {
        return tagline
}
```

And also way faster than the equivalent with a pure strongly typed, nested Swift Dictionaries.

```swift
if let movies = swDict["Movies"],
    metadata = movies["Die Hard 2"],
    tagline = metadata["TagLine"] {
        return tagline
}
```

Like, at least 2 times faster in both cases. Doh.

In the first case, there's a performance hit every time you convert between bridged classes. You can't just bridge back and forth from NSDictionary to Swift Dictionary. And bridging to the Swift class doesn't mean it just automatically gets faster because it's now a Swift class. Which leads me to the second case: Swift's Dictionary just isn't that fast compared to NSDictionary. "Blasphemy!" you cry. It's unfortunately true. Try it for yourself, it's way slower. I also benchmarked non-nested fetches (just a single valueForKey) and even there NSDictionary is much faster.

Bridges aren't what you think they are. They're quite different behind the scenes and if you have a function called a lot that uses Dictionaries, you'll unfortunately be better off with the legacy option for now.

Next up, benchmarking this in Swift 2.0.
