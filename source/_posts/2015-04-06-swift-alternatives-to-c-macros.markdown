---
layout: post
title: "swift alternatives to c macros (part i)"
date: 2015-04-06 22:05:57 +0100
comments: true
published: true
categories: iOS OSX Swift
---
Love them or hate them, macros are an integral part of C development, and are widely used throughout the Objective-C Cocoa and Cocoa Touch frameworks. If you've ever used `NSAssert`, then you've used a macro. (And if you haven't, then what have you been doing?!)

Now please bear with me... before all you macro-haters completely disregard this blog post, this is not another one of those posts that explains the trivial procedure of re-writing a function-like macro as a function in Swift. Nor will I show the Swift alternative to a macro definition of a primitive constant, as I've seen in other recent posts within the community.

In this short series, I'm going to look at some of the more creative uses of C macros and investigate how we could use some of Swift's advanced language features to accomplish the same results.

<!-- More -->

# Misuse of macros
I have come across a few blog posts recently which take various types of Objective-C macro and find alternative solutions in Swift. But every macro discussed could have been written as a function (or even worse declared as a constant) in Objective-C in the first place, so would be trivial to convert to Swift. I was going crazy staring at the screen in disbelief of people converting macros such as the following:

```objc
#define SQUARE_NUMBER(n) n * n
```

The blog post failed to point out that the Objective-C developer that wrote this macro in the first place should have been shot in the head with a [Nerf](http://nerf.hasbro.com/en-us) gun and sent back to junior school. There is so much wrong with that macro, I don't want to waste any more of my life discussing it.

I don't condone or encourage the use of macros in Objective-C where functions or constants could be used instead, and I don't think that any other iOS developer should do so either. And I certainly don't think the Objective-C community should be encouraging this by promoting or linking to this type of blog post.

There is never a need to write a function-like macro in Objective-C. If you're worried about performance (of calling out to a function), then there's no need. Cocoa provides the macros `NS_INLINE`, `CF_INLINE`, etc. which mean that your friend the compiler will decide whether it's appropriate to expand the function definition in-line. See the definition of the `CGRectMake` function for example. And there is certainly no need to `#define` primitive constants!

Yet even [Apple's documentation](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html#//apple_ref/doc/uid/TP40014216-CH8-XID_19) seems to condone the use of primitive constants being declared as a macro. By the time I read this, I was getting pretty fed up. Why is everyone, including Apple, encouraging the misuse of macros?!

Apple's example below should never have been written as a macro in the first place.
```objc
#define FADE_ANIMATION_DURATION 0.35
```
It provides no type-safety and it won't be available to interrogate in the debugger. There's a real danger this type of macro could be misused and lead to hard-to-diagnose bugs. In short, it's code smell that should be avoided at all times, with no excuses.

# So are macros ever acceptable?

Some people would probably argue the fact that it's never acceptable to use macros. I would encourage those people to read on because you might discover some cool tricks!

I wrote recently about KVC collection operators and [how to use them safely](/blog/2014/07/25/collection-operators-done-properly/). In that post, I discuss the use of macros to provide compile-time checking on the keys used in key-value coding.

Consider the following example interfaces:
```objc
@interface Account : NSObject
@property double balance;
@end

@interface Transaction : NSObject
@property Account *account;
@property double value;
@end
```
I can use KVC collection operators to simplify retrieval of transaction characteristicts from an array of `Transaction *` objects:

```objc
double minimumValue = [transactions valueForKeyPath:@"@min.value"];
double maximumAccountBalance = [transactions valueForKeyPath:@"@max.account.balance"];
NSArray *accounts = [transactions valueForKeyPath:@"unionOfObjects.account"];
NSArray *accounts = [transactions valueForKey:@"account"]; // same as above
```

I haven't used a macro yet! But I can use macros as defined in [my other post](/blog/2014/07/25/collection-operators-done-properly/) to perform these operations and get compile-time checking of the key paths with code-completion *and* syntax highlighting! (The highlighter on my blog doesn't display it as nicely as Xcode does.)

```objc
double minimumValue = DZLMinimum(transactions, Transaction *, value);
double maximumAccountBalance = DZLMaximum(transactions, Transaction *, account.balance);
NSArray *accounts = DZLUnionOfObjects(transactions, Transaction *, account);
```

Notice that the key paths are not enclosed in quotes. This is due to how they are expanded at compile time to be verified and then turned into strings to pass to `-valueForKeyPath:`.

So a macro that adds compile-time validation and code-completion to an otherwise crash-prone interface is, in my opinion, a decent example of when to use macros in Objective-C.

# How does this type of macro translate to Swift?

In the macros above, we pass the type expected in the array, namely `Transaction *` and pass the key path that we want to extract. In Swift, the type of objects in an array is mandatory when you declare the array, and we can extract information using the `map` or `reduce` functions.

For example, the above can be re-written in Swift as follows:

```swift
var minimumValue = transactions.map({$0.value}).reduce(Int.min, {max($0, $1)})
var maximumValue = transactions.map({$0.account.balance}).reduce(Int.max, {min($0, $1)})
var accounts = transactions.map {$0.account}
```

You can read up on these functions in [Apple's documentation](https://developer.apple.com/library/ios/documentation/General/Reference/SwiftStandardLibraryReference/Array.html). I also use the shorthand name of the first closure argument `$0`, which is also discussed in [Apple's documentation](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Closures.html).

But you could easily argue that the above is a little long-winded, in which case you could simplify it by extending `Array`. I've shown how to do this for the `minimum` below, but it could be easily modified for `maximum`, or `average`.

```objc
extension Array {
  func mapMinimum<U :Comparable>(transform: (T) -> U) -> U? {
    var mapped:Array<U> = self.map(transform)
    return mapped.count == 0 ? nil : mapped.reduce(mapped.first!, { max($0, $1) })
  }
}
```

In the above extension, I'm defining the `mapMinimum` function, which can be invoked on an array whose members all conform to the `Comparable` protocol (in order to be passed into the `max` function). It returns the minimum of the members of the array resulting from calling `transform(x)` on all members `x` of `self`. In other words, it performs the `map` function with the given closure, and then returns the minimum of the resulting array.

Having declared the same for maximum, I can now use them as follows:
```objc
var minimumValue = transactions.mapMinimum {$0.value}
var maximumAccountBalance = transactions.mapMaximum {$0.account.balance}
```

So the syntax is actually improved in Swift compared to the macros I defined in Objective-C. The key path is validated by the compiler, because it knows what types of object are contained in the `transactions` array.

# Next time...

I'll be investigating more of what I consider "valid" uses of macros and how we can achieve the same result using the advanced language features of Swift.

If you've enjoyed reading this article, please [follow me on twitter](http://twitter.com/dodsios) or subscribe to my [RSS feed](http://octopress.dev/atom.xml) for more of the same. I'm happy to discuss this or any other subject in more depth, so feel free to contact me!













