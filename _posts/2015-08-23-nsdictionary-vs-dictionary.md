---
layout: post
title: Swift's Dark Secret
comments: True
---

Since my last post I've been investigating different ways of accessing data in an NSDictionary in Swift and measuring the performance.

In that post, I thought that using an if/let chain that casts each nested dictionary individually caused a significant performance hit, and using an optional chain to access a nest value is much faster. That's not entirely correct: yes using an optional chain is much faster, but not for the reasons I thought. It turns out that it's what was being done inside the if/let chain, and the misuse of bridged classes, that was the culprit. Let me explain.

First of all, I wasn't building the test nested NSDictionary correctly. What I should have done was built a nested NSDictionary out of only NSDictionaries, like what happens when you load a plist. Instead I mistakenly created an NSDictionary of Swift Dictionaries. I thought that since NSDictionary bridges to Dictionary, they're basically the same thing under the hood, just accessed different ways.... Well, they're not.

If we create a pure nested NSDictionary, we can access nested values like this:

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

And also way faster than the equivalent with a pure strongly typed, nested Swift Dictionary.

```swift
if let movies = swDict["Movies"],
    metadata = movies["Die Hard 2"],
    tagline = metadata["TagLine"] {
        return tagline
}
```

Like, at least 2 times faster in both cases. Doh.

In the first case, there's a performance hit every time you convert between bridged classes. You can't just bridge back and forth from NSDictionary to Swift Dictionary. And bridging to the Swift class doesn't mean it just automatically gets faster because it's now a Swift class. Which leads me to what makes the second case also not performant: Swift's Dictionary just isn't that fast compared to NSDictionary. "Blasphemy!" you cry. It's unfortunately true. Try it for yourself, it's way slower. I also benchmarked non-nested fetches (just fetching a value from a single dictionary) and even there NSDictionary is much faster.

Bridges aren't what you think they are. They're quite different behind the scenes and if you have a function called a lot that uses Dictionaries, you'll unfortunately be better off with the legacy option for now.

Next up, benchmarking this in Swift 2.0.
