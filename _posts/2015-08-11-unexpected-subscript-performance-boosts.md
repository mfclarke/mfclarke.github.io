---
layout: post
title: Unexpected Performance Boost Using Subscripts Over Explicit Casting
---

I have a function that returns a specific nested dictionary. It looks like this:

```swift
private func fontDict() -> Dictionary<String, AnyObject>? {
  if let fonts = configDict["Fonts"] as? Dictionary<String, AnyObject> {
    return fonts[self.rawValue] as? Dictionary<String, AnyObject>
  }

  return nil
}
```

My view performance was lagging so I did a quick profile. This method was clocking in at 156ms on my main thread (it's called a lot). Not good.

Refactoring to this:

```swift
private func fontDict() -> Dictionary<String, AnyObject>? {
  return configDict["Fonts"]?[self.rawValue] as? Dictionary<String, AnyObject>
}
```

And the time drops to 36ms. Quite the performance boost!
