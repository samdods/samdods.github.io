---
layout: post
title: "Why I Love Swift's Protocol Extensions"
date: 2015-08-04 08:00:00 +0100
comments: true
categories: iOS OSX Swift
published: true

---

When protocol extensions were first mentioned at WWDC in June, they immediately stood out as a necessary feature. But it was only after I started playing around with them that I realised the true value these bring to the language.

<!-- More -->

## The problems in Objective-C that are somewhat solved by Swift

### The ability to "mix in" functionality to classes

When you want to extend a class in Objective-C, you would typically create a category on that class.

Say in my app I have defined the `Video`, `Photo` and `Song` classes, which have the shared superclass `MediaItem`.

Now I want to give each of these classes some functionality to make them `Shareable`. And I would expect this functionality to mostly be the same in each class.

Now you might suggest I implement the shareable functionality in their common superclass, and then somehow specify whether each one is shareable. But I might not want all future subclasses to inherit this functionality, and I may want to give this functionality to other classes that don't inherit from `MediaItem`. For example, I might want to make `UserProfile` shareable, so my users can share each other's profiles.

> With inheritance, you would typically follow the "is a" principle, meaning an object of your subclass type is an object of its superclass type and inherits common properties. For example, in real life, a `Dog` _is a_ `Mammal` so it would make sense to inherit from the `Mammal` class.
> 
> A `Dog` should also be capable of other functionality, for example being able to `run`. It doesn't make sense to implement that functionality in the `Mammal` class, because it would not apply to the `Whale`, which is also a subclass of `Mammal`.
> 
> It doesn't make sense to have a `Runner` class for `Dog` to inherit from, because a `Dog` isn't necessarily a `Runner`, so the "is a" principle doesn't apply. Being able to `run` is a capability rather than a property.

So how can I provide the `Shareable` functionality to my media classes without implementing it in each class individually? Well in Objective-C it simply isn't possible. Of course, you could add the shared functionality to a category on `NSObject`. But then (almost) all of your classes will inherit this functionality, which you don't want.

This is where Swift's protocol extensions come in handy. You could define the `Shareable` protocol like so:

```swift
protocol Shareable {}
extension Shareable {
  func share() {
    // common share functionality
  }
}
```

Now any class that you declare as conforming to the `Shareable` protocol will "adopt" the sharing functionality. For example:

```swift
class Photo: MediaItem, Shareable {
  // func share() adopted from Shareable protocol
}

class UserProfile: Shareable {
  // func share() adopted from Shareable protocol
}
```

If you are familiar with Ruby, you will see the resemblence to what is known as a mixin.

### The requirement to check `respondsToSelector:`

For aeons people have been trying to work around this problem, for example Peter Steinberger's [Smart Proxy Delegation](http://petersteinberger.com/blog/2013/smart-proxy-delegation/). But all such solutions are only work arounds.

We all know that pure Swift protocols don't support optional methods. And Swift objects don't support `respondsToSelector`. But with protocol extensions, none of this matters!

We can simplify what used to be five rather ugly lines of Objective-C...

```objc
BOOL shouldBegin = YES;
id<DataServiceDelegate> delegate = self.delegate;
if ([delegate respondsToSelector:@selector(dataServiceShouldBeginSync:)]) {
  shouldBegin = [delegate dataServiceShouldBeginSync:self];
}
```

... into a single line of Swift...

```swift Swift
let shouldBegin = delegate.dataServiceShouldBeginSync(self)
```

This is acheived by defining the default implementation of this methods in an extension to the protocol, as follows:

```swift
extension DataServiceDelegate {
  func dataServiceShouldBeginSync(dataService: DataService) -> Bool {
    return YES;
  }
}
```

The additional beauty of this is that it makes the default behaviour explicitly clear to the consumer of your interface. No longer do they have to hunt around in possibly out-of-date documentation to figure out what will happen if they don't implement this methods themselves. And the default behaviour isn't hidden in the implementation.

## The additional powers of protocol extensions

You can specify a "where" clause for your extension, so that only specific conforming classes adopt the functionality defined in the extension.

This allows us to define different default behaviour depending on the class that conforms to the protocol. For example, I could define the following protocol:

```swift
protocol Shareable {
  func share()
}
```

Now I can specify that any object that conforms to this protocol should use the default behaviour as defined:

```swift
extension Shareable {
  func share() {
    print("sharing something")
  }
}
```

And I can specify that all media items have a different default behaviour:

```swift
extension Shareable where Self : MediaItem {
  func share() {
    print("sharing a media item!")
  }
}
```

> Note that this is not the same as providing this implementation in the `MediaItem` class, because this doesn't mean all `MediaItem` objects are `Shareable`. Only subclasses that conform to the `Shareable` protocol will pick up this functionality.

This is demonstrated below:

```swift
class UserProfile : Shareable {}
class AudioTrack : MediaItem, Shareable {}
class Video : MediaItem {}

let userProfile = UserProfile()
userProfile.share()
// prints "sharing something"

let audio = AudioTrack()
audio.share()
// prints "sharing a media item!"

let video = Video()
video.share()
// results in a compiler error: value of type 'Video' has no member 'share'
```

This also means that you can refer to `self` in the default implementation and access the properties of `self` with the knowledge that it is of a particular type. For example, the `MediaItem`  might have a `copyrightTerms` property, so I can use it in my default implementation as follows:

```swift
extension Shareable where Self : MediaItem {
  func share() {
    if (self.copyrightTerms) {
      // present the copyright terms to the user, asking them to accept before continuing.
    } else {
      // complete share action without showing copyright terms.
    }
  }
}
```

## So are there any drawbacks?

Like with so many language features: with great power comes great responsibility. And even more importantly, comes the greater need to fully understand how something works.

The methods in a protocol extension can also be implemented in the class that conforms to the protocol. The implementation in the class overrides the "default" implementation in the protocol extension. What's more is that if the implementation is added to the class in the form of a class extension, then it too overrides the protocol extension implementation. This can get very confusing if you are defining different implementations using the "where" syntax.

As more people become accustomed to writing Swift, I'm sure the most common use for protocol extensions will be for the default implementation of "optional" delegate methods.

But for those that do use them as a kind of "mixin", I'd suggest erring on the side of caution and making your intent very clear.

## I hope you enjoyed reading

If you enjoyed this article, please [follow me on twitter](http://twitter.com/dodsios) or subscribe to my [RSS feed](http://octopress.dev/atom.xml) for more of the same. I’m happy to discuss this or any other subject in more depth, so feel free to contact me!

And if you'd like to join me at [The App Business](http://www.theappbusiness.com) to work on some awesome projects for the biggest clients in each industry, then please get in touch -- we're looking for great Swift and Objective-C developers.


