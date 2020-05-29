---
redirect_from: "/blog/2015/09/21/some-swift-best-practices/"
description: Article
layout: post
title: "Swift best Practices"
date: 2015-09-21 08:00:37 +0100
comments: true
categories: blog
published: true

---

I started learning Swift the day it was announced and read Apple's _The Swift Programming Language_ in the first couple of days. I began writing small components and playing around with Swift at the same time, but never felt it was mature enough to begin a big project with it.

That was until Swift 2.0 was announced in June. Since then I've been using it more and more and recently started writing a major project at [The App Business](www.theappbusiness.com) purely in Swift 2.0.

Read on for some of my observations -- some are obvious and common, but hopefully some will be new to most people. Let me know if you have any of your own that I could add to the list!

<!-- More -->

## Prefer `let` over `var`

Train your brain and your keyboard-bashing fingers to write `let` by default. There may be times when you know up-front you'll need a mutable variable, but I still recommend using `let` until you absolutely need to modify its value. This is the most obvious of my best practices, but it's super important, so still worth a mention.

## Prefer `private` access control

Again, this one is obvious, but still worth pointing out. It's always better to keep as much of your implementation `private`, which means it can only be accessed from the same source file.

## Prefer non-optionals

Optionals are great. This concept does exist in other languages (Scala, Haskell, etc.), but much respect has to go to Chris Lattner for bringing it to Swift.

However, as useful as they can be, I would still try to avoid them where possible. It leads to much cleaner code if you have variables and properties that you know hold a value, or functions that you can guarantee return a non-nil value.

For example, I would prefer a function to throw an exception if it is unable to return a non-nil value. Take the SDK's `NSJSONSerialization.JSONObjectWithData` function. It guarantees that if it returns anything, it will return a non-nil `AnyObject`, otherwise it throws an exception.

## Prefer `guard let` over `if let`

What can you do in an `if let` code block? I would _almost_ always opt for a `guard let` and return early. Early return tends to make the code that follows easier to read because you can guarantee the state of variables at that point. If you can't calculate a required value, then return early.

## Don't be afraid to `throw`

Instead of returning early, consider throwing an error, which is an alternative exit point from the function. Exception handling in Objective-C (`@try @catch`) was always possible, but frowned upon. In Swift, throwing errors is a fundamental concept and should be embraced.

I had a parsing function that returned an optional, returning `nil` if the data couldn't be parsed. This meant that the caller would have to check for `nil`. I refactored it to be a function that always returns a non-optional and `throws` an error if it can't do so (see above related to preferring non-optionals).

## Never `guard` against multiple conditions

Consider the following code:

```swift
guard let data = data,
          json = self.jsonFromData(data),
          authors = json["authors"] else {
  throw AuthorParserError
}
```

The above avoids the repetition of throwing the `AuthorParserError` in three places, but in my opinion it would be much better written as follows:

```swift
guard let data = data else {
  throw AuthorParserError
}
guard let json = self.jsonFromData(data) else {
  throw AuthorParserError
}
guard let authors = json["authors"] else {
  throw AuthorParserError
}
```

The above is better because you can now test each case individually. And you can be sure that you've covered each case, because Xcode 7's awesome code coverage facility will highlight any of the conditions that are not tested.

## Always inject dependencies, even if only for testing purposes

The great thing about Swift's optional function arguments is that you can dictate how something should ordinarily be used but at the same time open up your classes for easy testing - in particular mocking of dependencies.

If your class relies on something to handle network requests, for example, then why not pass that into the `init` function?

This will make testing a doddle, because you can mock the dependency, ensuring that you are only testing the functionality of that individual class. (Of course you _can_ write more integrated tests too.)

For example:

```swift
public init(requestDelegate: MyClassRequestDelegate = RequestManager()) {
  self.requestDelegate = requestDelegate
}
```

The dependency in the above example is actually a protocol, which makes testing very simple, because we just need to mock an object that conforms to the protocol.

(Some people might even demand that the dependency is always injected, instead of defining the defaul. But I feel the main purpose for this type of dependency injection is for easy testing. Unless you're building a library, your app will probably always use the same dependency, so it usually makes sense to have a default.)

## Always `typealias` completion handlers (where possible)

Despite [the syntax of closures being somewhat confusing](http://www.fuckingclosuresyntax.com), the syntax for type aliasing a closure is unquestionably simpler than the equivalent `typedef` in Objective-C. So always define a `typealias` for your completion handlers (unless they contain generics, in which case you can't).

```
typealias SomethingCompletion = (result: SomeType) -> Void
```

## Use `enum` to avoid ambiguity

Here I'm referring to a tuple or a set of completion handler arguments that may give rise to ambiguity.

Consider the following completion handler:

```
doSomething() { (output: NSData?, error: NSError?) in
  // need to check if we have output or error
}
```

> Note that I'm still using NSError as part of the completion handler, rather than throwing an exception, because you can't throw an exception asynchronously.

The above shows a completion handler which takes two optional values as its arguments. This goes against one of my previous points of "avoiding optionals" where possible.

What happens in the above if we have neither `output` nor `error`. Or what if we have both? Which do we handle?

You should be clear about the arguments to your completion handler, and you can use `enum` to help. For example, you could define the following:

```
enum Result<U> {
  case .Success(output: U)
  case .Failure(error: NSError)
}
```

Now we can rewrite the above use of the completion handler as:

```
doSomething() { result in
  switch (result) {
  case .Success(let output):
    // use output
  case .Failure(let error):
    // handle error
  }
}
```

This is absolutely clear now that if it was successful, there will be a non-nil output. And we even know the type of it, assuming the `doSomething` function specialises the generic `Result`. And likewise it is clear that if it failed there will be a non-nil error. And it can only succeed or fail, so there is no ambiguity.

## A trick for generic completion handling

Let's say you have a function declared in a protocol as follows:

```swift
protocol ServiceProvider {
  func provideService<U where U: AnyService>(completion: (output: U) -> Void)
}
```

When making a class conform to the `ServiceProvider` protocol (i.e. when implementing this function), as long as we know how to instantiate an object of type `U where U: AnyService`, then we can return an object of the correct type. (The `AnyService` protocol must provide a way of returning an instance.)

It is then up to the caller to dictate in the completion closure what type of object should be returned.

For example:

```
myOtherClass.provideService { (output: RoomService) in
  // do something with the RoomService
}
```

The implementation of the `provideService` function doesn't need to know anything about the `RoomService` class. It simply has to conform to `AnyService`, which allows it to be instantiated somehow.

What I really like about this, is that the caller can get back whatever they want back. The caller simply says, "in the completion handler, I want an instance of `RoomService`" and the implementation of the function knows how to deliver.

This is in stark contrast with Objective-C, where we would define a completion block argument of type `id` and the caller would tell the compiler what type of object they _expect_ back when they implement the completion handler. But in Swift, we are _guaranteed_ an object of this type. What's more, we are given this guarantee at compile time.

## Closure best practices

I try to keep closures as concise as possible.

* If a closure returns `Void`, I don't write the return type
* If the type of the object can be inferred by the compiler, then don't write the type
* If the closure only has a single argument, then don't put it in parentheses
* Always make the closure the last argument to a function
* Always trail the closure when implementing
* Don't write redundant parentheses that the compiler doesn't require (for example, if the function only has one argument and that argument is the closure)

These are largely up for debate, but this is the way I'm writing things right now. For example:

```
myObject.doSomething { output in
  // do something with the output
}
```

As opposed to:

```
myObject.doSomething() { (output: NSData?) -> Void in
  // do something with the output
}
```

## I hope you enjoyed reading

If you enjoyed this article, please [follow me on twitter](http://twitter.com/dodsios) or subscribe to my [RSS feed](http://octopress.dev/atom.xml) for more of the same. I’m happy to discuss this or any other subject in more depth, so feel free to contact me!

And if you'd like to join me at [The App Business](http://www.theappbusiness.com) to work on some awesome projects for the biggest clients in each industry, then please get in touch -- we're looking for great Swift and Objective-C developers.

