---
layout: post
title: "Delegate Pattern in Swift"
date: 2016-12-01 21:39:37 +0000
comments: true
categories: iOS OSX Swift Best-Practices
published: true

---

# Delegate Pattern in Swift

I started a long and mostly one-sided debate in the office last week, by suggesting that we should _not_ change our delegate method signatures to conform to the Swift principle of **omitting needless words**.

My side of the debate was that we should follow the convention of prefixing delegate methods with the name of the delegating class, as follows:

```swift
func scrollViewDidScroll(_ scrollView: UIScrollView)
```

The other side of the debate was, “Come on, man, this is Swift 3. Those words are not needed, so omit them! It’s more clear at the point of use!” And this side of the debate suggested we adopt the following naming convention:

```swift
func didScroll(in scrollView: UIScrollView)
```

In this article, I’ll rationalise my side of the debate, but also ask the question: is delegation even the right pattern?

<!-- More -->

## Clarity at the point of use

A large part of writing an API is to allow others to invoke our method implementations. We provide an interface to our code. And we should make is as simple as possible to interface with. We’ll write a method once, but many people will call our method, possibly many times. So clarity at the point of use is more important than at the point of declaration. This is also referred to as clarity at the call site.

Delegate methods are different to typical methods in our interface. By declaring a delegate protocol, we are writing a contract. This contract defines an interface to code that we expect others to implement. The roles are reversed for delegate methods. Instead of inviting someone to invoke a method that we implemented, we’re asking them to implement the method, so that we can call it. So the guidelines should be reversed too. We will call the method once, but many people will implement that method, possibly many times. The “point of use” now refers to where the method is implemented by those using our API. I’ll refer to this as the implementation site.

The most important thing is still clarity at the point of use, only that point of use is now the implementation site. And the most important thing to ensure clarity around is that the method "belongs to" our API. That is to say, "you implement it, but don't call it. We'll call it." The best way to make this clear is to prefix the method with the name of the delegating class.

_As as aside, even if the guidelines were not reversed, I still don't agree that `didScroll(in:)` is clearer at the call site. `delegate?.didScroll(in: self)` is misleading. It wasn't the delegate that scrolled in self – it was self that scrolled!_

## Passing a reference to the sender

While this debate was taking place, it was also touted that you needn't pass the reference to the sender. "If you don't need it, don't send it!"

Actually, the delegate pattern is defined such that the delegate can implement the method on behalf of the delegating class, with the full original context of the sender. This is achieved in Swift by sending a reference to `self`. And when defining an API for others to consume, one should stick to the definition of the delegate pattern by providing the original context.

Moreover, you risk taking brevity too far and instead of having:

```swift
func didScroll(in scrollView: UIScrollView)
```

you'll end up with just:

```swift
func didScroll()
```

And where's the sense in that?!

A lot of people are suggesting that Apple just haven't updated their [Coding Guidelines for Delegate Methods](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CodingGuidelines/Articles/NamingMethods.html#//apple_ref/doc/uid/20001282-1001803-BCIDAIJE) since Objective-C. And if they did, perhaps they would review it with Swift's [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/) in mind.

However, I disagree again. Apple provided documentation on [Delegation](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html#//apple_ref/doc/uid/TP40014097-CH25-ID276) in the book _The Swift Programming Language (updated for Swift 3.0.1)_. It doesn't explicitly state that a reference to the sender must be passed as an argument, but they do follow that convention in all of their examples. They also (almost) follow the convention of prefixing all the delegate methods with the name of the delegating class. (Although in their examples, it's actually a delegating protocol type `DiceGame` and the delegate methods each have the consistent prefix `game`.)

## Following convention

Conventions are there to be followed. But rules are there to be broken, right? Otherwise how would we innovate? I agree with this, but I don't agree with breaking convention to end up with something that's _less_ clear.

In [Soroush Khanlou](https://twitter.com/khanlou)'s article about [Swifty Delegates](http://khanlou.com/2016/09/swifty-delegates/), he suggests that in an ideal future Apple would update UIKit to follow a more 'Swifty' delegate method signature. UIKit already strays from convention with the `UITableViewDataSource` method `numberOfSections(in:)`.

While I respect Soroush's opinion, and have thoroughly enjoyed many of his articles, I disagree that this is the future. I actually think UIKit would be better if anomolies like this were corrected. So this example would become `tableViewNumberOfSections(_:)`.

The prefix is there to distinguish between other methods that the delegate might want to implement. Apple is delegating this functionality to the consumer of its API. They are asking the consumer to implement this method. I don't want to implement it, but I have to, because Apple expects me to. And as such, I'd prefer it to have the `tableView` prefix, so I know – and others after me know – that this method belongs to the delegating class, and it's not just some method I've implemented for my own benefit.

But of course, that's just my opinion. I've expressed my will and desire.

But what is ultimately much more important, is this:

If you want to later query the number of sections in the table, from your delegate (say, a view controller), are you going to call:

```swift
self.numberOfSections(in: tableView) // Swifty-style delegate method implemented on the view controller
```

or the following?

```swift
self.tableView.numberOfSections
```

I'd recommend the latter, although with the number of sections it's _probably_ safe to call either. But what if you want to find the cell for a given row? Assuming UIKit had taken Soroush's advice, then which of the following would you call?

```swift
self.cellForRow(at: indexPath, in: tableView) // Swifty-style delegate method implemented on the view controller
```

or

```swift
self.tableView.cellForRow(at: indexPath)
```

It might be tempting to call the former because you start typing `cell...` and auto-complete serves it to you on a plate. But my advice would be to call the latter, given that only the _delegating class_ should ever call the _delegate method_.

But wouldn't it be really confusing if those `UITableViewDataSource` methods were really declared as Soroush suggested? And if we all stray from convention – to make our delegate method signatures more 'Swifty' – then we risk making our APIs confusing. The prefix convention highlights that this method should only be called by the delegating class.

## But is delegation even the correct pattern?

If you strictly follow the [guidelines laid out by the objc.io guys](https://www.objc.io/issues/7-foundation/communication-patterns/), then you might find yourself using delegation a lot less. But their article isn't new, so is the delegate pattern still overused? Probably, yes. But then if we're drawing inspiration from UIKit et al, then it's obvious why.

But blocks weren't present at the birth of UIKit. They came on the scene with iOS 4, and quickly gained popularity with many open source libraries extending UIKit to provide block-based APIs. Blocks, or now _closures_, may be much more appropriate in many of the places the delegate pattern is used.

The dictionary definition of the verb _to delegate_ is to:

> entrust a task or responsibility to another entity, typically one who is less senior than oneself.

Now, I'm not going to get into a debate about who's more senior: a view controller or a table view. But the important bit is that delegation is _entrusting a task_ to another entity. It's _not_ about notifying!

Some examples that came up during our debate were similar to this:

```swift
func myCellDidTapAdd(_ cell: MyCell)
```

or

```
func didTapAdd(on cell: MyCell)
```

This is a prime example of when a closure would solve the problem better than the delegate pattern.

### Closures

A closure may be better here because you'll typically have access to the cell's index path at the point at which you set the closure. The index path is much more useful than the cell itself, which would be passed to the delegate method implementation. This is because you're likely to look up an item from your data source using its index. It avoids having to call `tableView.indexPath(for: cell)` and guarding against a `nil` index path being returned. And within the closure you can call your own method, which _you've_ implemented because _you_ want it to do something, with the arguments _you_ need, rather than it being dictated by a delegating class that you _must_ implement it.

### Target/Action

And target/action might be better in many places too. Target/action is a great mechanism, which was vastly improved by the addition of `#selector` in Swift 2.2. If it's a simple case of delegating that something was tapped, why not make it a `UIControl` subclass and hook it up via target/action?

I've seen so many times a custom view with a `UITapGestureRecognizer`, where the method for the tap delegates its handling to the custom view's delegate. Have you ever seen something like this?

```swift
@IBAction private func handleTap(tap: UITapGestureRecognizer) {
    delegate?.myCustomViewWasTapped(self)
}
```

Why not try the following?

```swift
@IBAction private func handleTap(tap: UITapGestureRecognizer) {
    sendActions(for: .touchUpInside)
}
```

Now you can either set up the target/action in code, or hook it up in interface builder if appropriate.

And, as mentioned above with closures, the method that's implemented by the consumer can be whatever they choose. Both with closures and target/action, we allow the consumer of our API to write whatever methods they please, and they will still be "notified" by our object.

This means as a consumer we can hook the API into a method like `dismissCustomView()` instead of having to implement the delegate method `customViewDidTapDismiss(_:)`. This also means we don't need a separate method if we want to call this from somewhere else in our own code.

Furthermore, with target/action, it's the receiver that decides whether or not they need a reference to the sender. And if they want it, they've got it. And if they don't, they can implement a method that takes no arguments.


## Conclusion

I do think delegate methods are overused, but if we continue to use them then I think we should stick to convention. Mainly to avoid confusion, but also out of respect for the consumers of ones API – even if those consumers are only ones colleagues, or even oneself!

So when is it right to use the delegate pattern?

Typically I find that if you want something returned, then the delegate pattern is probably a good shout. Say, if you want to check if you should do something, then you'll expect a `Bool` to be returned. And this makes sense with the definition of delegation too. You're entrusting someone else to provide some information that you need.

And I'd suggest avoiding the delegate pattern if you're simply notifying another object of an action.

But of course, there are no hard and fast rules, and that's what's so great about programming – we can always find better ways of doing things, and [we don't need to get it right first time](https://8thlight.com/blog/daniel-irvine/2016/11/11/perfect-code-is-an-illusion.html?utm_campaign=iOS%2BDev%2BWeekly&utm_medium=email&utm_source=iOS_Dev_Weekly_Issue_277)!

And if we continue to ask questions like this, then we continue to improve as developers. And surely that's the most important thing.

Thanks for reading!

_This article ended up being a bit longer than I'd planned, so seriously, thanks for reading! If you're interested in continuous improvement and beating things up to find better solutions, then you might be interested in joining a great team at [The App Business](http://www.theappbusiness.com)._

_If you enjoyed this article, please [follow me on twitter](http://twitter.com/dodsios) or subscribe to my [RSS feed](http://octopress.dev/atom.xml) for more of the same._
