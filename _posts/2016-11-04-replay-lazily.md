---
layout: post
title: Replay Lazily for fun and profit (and sensible data fetching, prefetching and caching)
comments: True
---

Today I want to show off a very handy operator in Reactive Cocoa called `replayLazily`. This little guy (man I wish there was a sloth emoji, it would be so perfect) will perform the work once and then hold onto the resulting value, so when someone else calls `start` on him, he can just replay the value instead of performing the work again. Like the lazy sloth he is.

But, and here's the cool part, if while doing the work you call `start` on him again, he won't actually `start` again. He'll wait until the work is finished and then just replay it for you. Great for those slower network requests, or for when your user has slow connectivity.

Or.... prefetching! On app start you can call `start` on these guys and let them fetch, all the while knowing that if they are still waiting for a response when the user enters the screen the fetch was for, it won't fire off a second fetch, it will just wait for the value. Or, if the prefetch completed before the user got there, it will be replayed instantly when they hit that screen. No logic to worry about, just 1 simple `SignalProducer` with a `.replayLazily(upTo: 1)` on the end 🎸

```
class DataProvider {
    static var producer = DataFetcher.fetch().replayLazily(upTo: 1)
}

// Prefetch as early as possible
DataProvider.producer.start()

// Use it normally when you need it
myClassThatNeedsData.dataProperty <~ DataProvider.producer

// Or if your app isn't fully reactive
DataProvider.producer.startWithValues { values in
    // do something cool with said values
}
```

Yep, that's your prefetching data store right there. Mindblown.gif.
