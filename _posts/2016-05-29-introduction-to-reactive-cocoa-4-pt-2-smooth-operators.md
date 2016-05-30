---
layout: post
title: Introduction to Reactive Cocoa 4 (pt 2) - Smooth Operators
comments: True
---

Before we begin pt 2 of the **ELI5 idiots guide to [Reactive Cocoa 4](https://github.com/reactivecocoa/reactivecocoa) for dummies who are [fluent in Swift 2](https://www.objc.io/books/advanced-swift/) but know nothing about any version of Reactive Cocoa (though maybe know a little bit of [what FRP is all about](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754))** (or TEIGRC4FDWAFS2BKNAAVRC for short), I'd like you to press play on the following video:

<a class="embedly-card" href="http://www.vevo.com/watch/sade/smooth-operator-(official-video)/GB1100800414">Smooth Operator (Official Video) - Sade</a>
<script async src="//cdn.embedly.com/widgets/platform.js" charset="UTF-8"></script>

## Multi Signal Structures

### Transforming Values with Operators

So far, everything we've done with ```Signal```s has been pretty basic. And while there is some real world use for them already, it's still not useful enough to start bowing down before the FRP gods. But you see, this is by design. We have these really simple blocks that we use to build complex data flow structures that are expressed very concisely and neatly. How do we join these blocks together? *Operator*s.

Operators transform the ```Value```s from ```Signal```s for different uses and contexts. When you transform a ```Signal```, ReactiveCocoa gives you a new ```Signal``` which fires the transformed ```Event```s, so they can be chained. This is very much like the functions Swift gives us for ```SequenceType```s: ```filter```, ```map```, ```flatMap``` etc. As you already know (and if you don't, get ready for the 💣), these apply to a ```SequenceType``` and return a new object of the same ```SequenceType```, so they can be chained. In other words, if you ```filter``` an ```Array```, you get a filtered ```Array``` in return, and you can directly ```map``` or whatever the result without assigning to a new var.

```
let longNamesCount = users
  .map { $0.name }
  .filter { $0.characters.count > 10 }
  .count
```

Operators function in the same way - chain them together to your hearts content. This lets you construct complex flow structures in a very concise way, all in the one place.

A simple demonstration of an operator in action: Say you have a ```Signal``` that has ```String``` values on it. Now, whenever that ```Signal``` has a ```String``` on it that contains the word "dog", you want to be notified.

Here's the wrong way to do it (but the only way we know how at this stage):

> ![Example 1](/images/rac4-2/example1.jpg)

Seems reasonable? Well... not quite. I mean it works, but it doesn't allow for any extensibility. For example, how do we notify more than one object that the string contains the word dog? Our ```observeNext``` closure (or the ```stringContainsDog``` function) needs to know about any object that needs to be notified. That sucks.

And what about other scenarios, like when a string contains "dog" and "cat"? Do we need a new external function for this? What if a different object needs to be notified on the new scenario? Here comes the giant flying spaghetti code monster...

What if I told you Reactive Cocoa has your back? 🕶

> ![Example 1](/images/rac4-2/example2.jpg)

Did we just solve each one of those hairy problems in one go? In the same number of lines?? 😮

Say hello to *operator*s.

Here we're using the ```filter``` *operator*. Generally speaking, this *operator* receives *Next* ```Event```s from the ```Signal``` it's attached to, and filters their ```Value```s using the predicate closure given. So in our case, take the ```String```s that come in from the ```stringSignal``` and filter them by "contains 'dog'". Notice how this logic moves out of our ```observeNext```.

This isn't the whole picture though. *Operator*s actually return brand new ```Signal```s that fire *Next* ```Event```s using their transform. So in the case of ```filter```, it fires when the predicate returns ```true```.

This is truly awesome for a few reasons, but first of all it means we can observe this new ```Signal``` to be notified when the string contains a dog, but not touch the behaviour of the ```Signal``` we're ```filter```ing. ```Signal```s are immutable, and if we don't introduce any side effects then they **always** do the same thing. Hey look, your code just got way easier to read and understand!

With *operator*s, you can not only set up very simple flows like the above, but also very complex flows that are still easy to understand. And all parts of the flow are observable by anything. Woah 🎇

In fact, we can actually make this flow even more concise by just chaining it all together!

> ![Example 3](/images/rac4-2/example3.jpg)

So nice. And because the return types of *operator*s vs ```observe``` functions force the chain the follow the same pattern every time, we can reason about the flow easily. Create, transform, observe. ```Event```s start at the top and flow to the bottom. They don't jump between different objects or files or anything like that (unless you want them to).

So, what about the extra notification, when the string contains "dog" and "cat"?

> ![Example 4](/images/rac4-2/example4.jpg)

It's too easy right? No side effects, no coupling, easy extension.

What if we don't really want to get notified per se, but just want to replace the word "dog" with a 🐶 emoji?

> ![Example 5](/images/rac4-2/example5.jpg)

Pretty simple - we use the ```map``` *operator* which has the exact same "return a ```Signal``` that fires transformed ```Event```s" characteristic. So, our ```map``` creates a new ```Signal``` that fires *Next* ```Event```s with the word "dog" replaced with 🐶. We also just inserted it into our chain, no bother.

What happens if a ```Signal``` sends a *Failed* ```Event```?

> ![Example 6](/images/rac4-2/example6.jpg)

You can see that the cat ruined all the fun 😠. The *Failed* ```Event``` caused the first ```Signal``` to stop before James had his "dog" transformed into emoji form. So the *Failed* ```Event``` halted the flow of ```Event```s.

Also notice that so our ```Signal``` can throw a ```CatError```, we have to specify this when creating the ```Signal```. Strong typing and all that.

That was kind of a contrived example though - we should automatically fail rather than tell the ```Signal``` to fail. The string "Wendy has a cat" should automatically throw the ```.WeHaveACat``` *Failed* ```Event``` right? *Operator*s to the rescue:

> ![Example 7](/images/rac4-2/example7.jpg)

We can use the ```attempt``` *operator* to check for the error case and throw if it's met. Even though ```Success``` doesn't pass any value, it indicates that all is well and ```Event```s can continue down the chain untouched. Then it's just business as usual.

Notice how all the logic for transforming our ```String```s is all in the same place, including error handling? And it's totally decoupled from anything that needs to react to the transformed ```String```s or errors. Can I get a hell yeah?  

In the next installment we introduce the final piece of the puzzle: ```SignalProducer```s. Until then! 👋
