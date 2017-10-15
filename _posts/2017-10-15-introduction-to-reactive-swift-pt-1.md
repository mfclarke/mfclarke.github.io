---
layout: post
title: Introduction to ReactiveSwift (pt 1)
comments: True
---

[Reactive Swift](https://github.com/reactivecocoa/reactiveswift) is a [Functional Reactive Programming](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754) framework for Swift. It’s [awesome](https://github.com/kickstarter/ios-oss). It can make your code declarative, [simple](https://www.infoq.com/presentations/Simple-Made-Easy) and way less stateful. It can handle things that normally need a bunch of `if`s and `bool` state, in just a handful of lines. It’s incredibly concise. I could honestly go on and on, but you’re probably reading this because you know it’s `awesome` and you want to get into it.

Well it’s your lucky day, because this series of posts is designed to ease you into FRP with ReactiveSwift, from the perspective of regular iOS development. I’m not going to start with a reactive login screen (FRP’s TodoMVC). We’re going to start with basic primitives and build up from there. Hopefully after reading these posts, you’ll skip the usual first step of the FRP n00b, which is to `reactive allthethings!` end up with a convoluted stinking pile of 💩 and then do a rewrite.

So let’s get started. I’m gonna to assume you’re [fluent in Swift](https://www.objc.io/books/advanced-swift/), you have a clone of the [Reactive Swift repo](https://github.com/reactivecocoa/reactiveswift), and you have the playground fired up and ready for action.

## Signals, Events, Values, say what?
Let’s say you want to send a `String` somewhere. In ReactiveSwift speak, that `String` is a value. This value is sent as an `Event` using an `Observer`, which emits it on a `Signal`. Anything interested in this `String` can connect to the other end of this `Signal` to receive it.

`String => (Observer => Signal) => YourStringReceiver`

At the moment it kinda seems like a regular delegate pattern with different nomenclature.  Let’s dig a little deeper. So, `Observer` and `Signal` go hand in hand. All `Signal`s have an `Observer` that emits the events through it, either implicitly or explicitly. Let’s create the pair explicitly using `Signal`’s `pipe()` convenience constructor:

```
let pipe = Signal<String, NoError>.pipe()
```

This returns a tuple, giving the tag `input` to the `Observer` and `output` to the `Signal`. `Signal`s and `Observer`s are generics, and in this case we specialise them to work with `String` values and no error handling, for now. More on errors later.

Now we have the pipe, we can observe values sent through the `Signal` by attaching a closure to it:

```
pipe.output.observeValues { value in
  print("🚙 My other car is a \(value)")
}
```

Now, every time an `String` is emitted on the `Signal` this `observeValues` closure is fired with it as the `value`.

Finally, to actually emit our `String`, we send it to the `Observer` object:

```
pipe.input.send(value: "DeLorean")
```

The `Observer` emits `DeLorean` on the `Signal` as a value `Event` (more on `Event`s later), arriving at our observation closure. The direct result of the above line being executed is the `observeValues` closure being fired once with `DeLorean` as the value.

All together now:

```
let pipe = Signal<String, NoError>.pipe()

pipe.output.observeValues { value in
  print("🚙 My other car is a \(value)")
}

pipe.input.send(value: "DeLorean")
```

Copy/paste this into the sandbox playground and play around with it. Try extra `pipe.input.send` lines, changing the value sent for each. You’ll see in the debug console that the block is fired exactly once with each value received in the order we sent them.

So one of the cool things about `Signal`s is that you can attach any number of closures. That is to say, they can be `observe`d by more than one thing. So the value of one `send(value:)` can be received and used in many different ways and contexts.

```
let pipe = Signal<String, NoError>.pipe()

pipe.output.observeValues { value in
  print("I got 99 problems but a \(value) ain't one")
}

pipe.output.observeValues { value in
  UIPasteboard.general.string = value
}

pipe.output.observeValues { value in
  UserDefaults.standard.set(value: value, forKey: "JustSummerThings")
}

pipe.input.send(value: "🏖")
```

Neat. But what if we want to stop observing a `Signal`? In other words, disconnect a single closure. In a normal delegate pattern situation we’d simply `nil` the `.delegate`. In a multicast delegate situation we’d have a `func` to call to remove an individual delegate. In the world of ReactiveSwift where there is no distinct object but rather a closure, we need a more fine grained construct. This is where `Disposable` comes in. Each time you observe a `Signal`, you get a `Disposable` in return.

```
let pipe = Signal<String, NoError>.pipe()

let problemsDisposable = pipe.output.observeValues { value in
  print("I got 99 problems but a \(value) ain't one")
}

let beersDisposable = pipe.output.observeValues { value in
  print("The \(value) is my favourite place to sling code")
}

pipe.input.send(value: "🏖")
problemsDisposable.dispose()
pipe.input.send(value: "🗽")
```

When you call `dispose()` on a `Disposable` it disconnects the observation, so the closure no longer fires.

Already this is pretty cool and we’re barely scratching the surface. We basically have a multicast delegate pattern in a handful of lines, without custom protocols 😎

## A Series of Unfortunate Events
But there’s more to `Signal`s than just transmitting values. They can also fail, or be interrupted, or complete. The `Event` `enum` is used to represent all these events. When you `.observe` a `Signal`, you’re really observing the `Event`s being emitted on that `Signal`. The `.observeValues` is a convenience which allows us to only pay attention to the `Event.value` events - in our case above, the `String` values. Let’s take a look at the other `Event` `case`s.

Firstly, `.failed`. Funnily enough, this is used to represent something that has gone wrong 🙅‍♂️. This has an `Error` type associated value. This gives us strongly typed and declarative error messaging. But there’s more to it. If a `Signal` receives a `.failed` `Event`, it will stop emitting any further `Event`s. So this is used if we encounter a situation where we cannot continue and need to inform anything observing the `Signal` what went wrong.

```
enum EmojiError: Error {
  case nonLyrical
}

let pipe = Signal<String, EmojiError>.pipe()

pipe.output.observe { event in
  switch event {
  case let .value(value):
    print("I got 99 problems but a \(value) ain't one")
  case let .failed(error) where error == .nonLyrical:
    print("Bro, do you even Jay Z?")
  default:
    break
  }
}

pipe.input.send(value: "🏖")
pipe.input.send(value: "🍺")
pipe.input.send(error: .nonLyrical)
pipe.input.send(value: "🏖")
```

Notice that the last value `"🏖"` didn't send, because the `Signal` had received `.failed` . Perfect. Sidenote: normally we would use an operator to detect that `"🍺"` is the wrong emoji and automatically fail. More on this later.

Next we have `.completed`. This also causes the `Signal` to stop, but without an `Error`. It is generally used to indicate success - the `Signal` has finished without issue and we don't need it anymore.

```
let pipe = Signal<String, NoError>.pipe()

pipe.output.observe { event in
  switch event {
  case let .value(value):
    print(value)
  case let .completed:
    print("🤗")
  default:
    break
  }
}

print("Hey barkeep, mix me something good")
pipe.input.send(value: "2 shots of Campari")
pipe.input.send(value: "Ice cubes")
pipe.input.send(value: "Soda")
pipe.input.send(value: "2 lime wedges")
pipe.input.sendCompleted()
```

Finally there’s `.interrupted`, which is very similar to `.completed` but is used to simply stop the `Signal` rather than indicate success. This might not seem useful right now, but when we get into operators and chaining it will become clearer. As an example, pretend a `Signal` could be used to process an array of images[^1], but during processing the user leaves the screen. The `Signal` should stop work, but it didn’t complete nor did it error. It has been interrupted. This way, the app can clean up anything being used for image processing but leave out the success path that would have shown an alert or something. Without any `bool`s or `if`s or external state 💯

In each case above where the `Signal` stops (`.failed`, `.completed`, and `.interrupted`), it will not emit any further `Event`s even if its `Observer` attempts to. So if we're clumsy with our code and try to send a `.value` `Event` after a `Signal` has completed, it won't send - the `Signal` is already stopped. It also won’t throw an exception or anything.

And one last note about manually created `Signal`s (like we do above with `.pipe`): if they still have active observers (closures that haven’t been disposed), they won’t `deinit` until they receive `.completed`, `.failed` or `.interrupted`. It’s really important to check your `Signal`s `deinit` correctly when you’re starting out.

Phew, quite a lot to take in I’m sure. So that’s all for now. Stay tuned for the next post, where we add operators to the mix and build a small playground to search for party parrot gifs. Of course. See you then! 👋
