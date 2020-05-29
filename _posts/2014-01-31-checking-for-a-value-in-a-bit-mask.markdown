---
redirect_from: "/blog/2014/02/01/checking-for-a-value-in-a-bit-mask/"
description: Article
layout: post
title: "Enumerated Types and Checking Their Values"
date: 2014-02-01 11:06:27 +0000
comments: true
categories: blog
---
Enumerated types are very useful and widely-used. Foundation framework gives us two macros to help give definition to an enumerated type: [NS_ENUM and NS_OPTIONS](http://nshipster.com/ns_enum-ns_options/). But their intended use -- and correct use -- isn't immediately obvious.

First a bit of background: Enumerated types are a part of the ANSI C standard and, if I was a gambler, I would bet my entire future life's earnings on the fact that every single iOS or OS X programmer has encountered them and is very likely to use them daily, even if they are unaware of it. They are defined in a similar way to a struct and it's very common to `typedef` an enumerated type for easier use.

In this post I will discuss the Objective-C macros `NS_ENUM` and `NS_OPTIONS`, their origins and, most importantly, the correct way to check the value of a variable of an enumerated type.

<!-- more -->

# Enumerated Types
An enumerated type is a data type that consists of named integer values. They are frequently used to represent options. For example, you could define the following enumerated type:

```objc
enum AnimationOptions {
 // 1.
  AllowUserInteraction      = 1 << 0,  //   1
  Repeat                    = 1 << 1,  //   2
  Autoreverse               = 1 << 2,  //   4

 // 2.
  TransitionFlipFromLeft    = 1 << 4,  //  16
  TransitionFlipFromRight   = 2 << 4,  //  32
  TransitionCurlUp          = 3 << 4,  //  48
  TransitionCurlDown        = 4 << 4,  //  64
  TransitionCrossDissolve   = 5 << 4   //  80
};
```

1. The first 3 options defined in the above enumerated type, 1, 2 and 4 are distinct bits, i.e. `0001`, `0010` and `0100` respectively. This allows none, one, some or all of these options to be used at the same time. For example `0110` (i.e. 6) means the animation will repeat and auto-reverse.

2. The second set of constants are not distinct bits. This is intentional so that only a single transition type can be used. If I try to set both the `TransitionFlipFromRight` option and the `TransitionCurlUp` option at the same time, I'll end up with the `TransitionCrossDissolve` option (because 32 + 48 = 80).

You could `typedef` this enumerated type and use it as follows:

```
typedef enum AnimationOptions AnimationOptions;

@interface MyAnimation : NSObject
- (instancetype)animationWithOptions:(AnimationOptions)options;
@end
```

# Sized Enumerated Types

C++ extends the ANSI C definition of `enum` allowing you to specify the size of your enumerated type. You can use any integer type, for example `int`, `char` or `unsigned long`.

We could inline `typedef` and specify a size for the the `AnimationOptions` enumerated type as follows:

```
typedef enum AnimationOptions : unsigned int {
  AllowUserInteraction    = 1,
 // etc.
} AnimationOptions;
```

# NS_ENUM and NS_OPTIONS

Foundation introduced these macros in OS X 10.8 and iOS 6. Since then they have been the preferred way to declare enumerated types, because each macro implies the intended use of its underlying integer values. They are both defined in exactly the same way. They are identical to each other in all ways but one -- their intended use:

* Use `NS_ENUM` to describe a type of variable that may only be set to a single option at any time. An example is `UIInterfaceOrientation`, because the device can only be held in one orientation at any time.
* Use `NS_OPTIONS` to describe a type of variable that may contain multiple options at the same time. Some of these options may apply at the same time as others; some may not be used in conjunction with any others in the same set. This is referred to as a bitmask, where each bit--or set of bits--represents an option. An example is `UIInterfaceOrientationMask` -- this is used when multiple orientation options may be stored in a single variable, such as `unsigned int supportedInterfaceOptions` -- we can support more than one interface orientation at the same time.

Both of these macros are defined in `<Foundation/NSObjCRuntime.h>`, and what they boil down to is the following:

```
#define NS_OPTIONS(_type, _name) enum _name : _type _name; enum _name : _type
```

This isn't immediately obvious in Xcode because it doens't allow you to "Jump to Definition". But it is evident when inspecting the result of the preprocessor. The following is an example of using `NS_OPTIONS`:

```
typedef NS_OPTIONS(NSUInteger, AnimationOptions) {
  AllowUserInteraction = 1,
 // etc.
};
```
The above example results in the following output from the preprocessor:
```
typedef enum AnimationOptions : NSUInteger AnimationOptions; enum AnimationOptions : NSUInteger {
  AllowUserInteraction = 1,
 // etc.
};
```

> Note it is up to you to enter `typedef` before the `NS_ENUM` or `NS_OPTIONS` macro.

Because `NS_OPTIONS` can hold multiple options at the same time, variables of these types can be set with multiple values using the bitwise OR operator `|`, as follows:

```
AnimationOptions options = Repeat | Autoreverse | TransitionCrossDissolve;
```

> Note that in C a variable of an enumerated type can hold a value that is not defined in its underlying enumerated type. With Objective-C (and more specifically the GDB and LLDB compilers), we are warned at compile-time if we set the value of an `enum`-type variable to anything other than what is defined in its underlying enumerated type definition.

In `NS_OPTIONS`, a set of bits is used as one option when the option has multiple possible values. In `UIViewAnimationOptions` bit 3 specifies that the animation should repeat indefinitely. This is a binary option and can only be on or off. However, bits 16-19 are used to hold the animation curve option, which can be one--but only one--of several options. This is like an NS_ENUM (one option at a time) embedded within the options of an `NS_OPTIONS` type.

# Checking the Value of an Enumerated Type Variable

## NS_ENUM
Checking the value of an `NS_ENUM`-type variable is simple. It is an integer value and it can only contain one option at a time, so it can be checked for equality like any other integer, for example:
```
UIInterfaceOrientation orientation = device.orientation;
BOOL isPortrait = (orientation == UIDeviceOrientationPortrait);
```

## NS_OPTIONS
As mentioned above, `NS_OPTIONS` is intended to hold a list of options that may be used at the same time. Therefore variables of this type are referred to as bitmasks.

Checking the value of a bitmask can't be done by checking for equality, because multiple options are stored in the same variable by setting different bits. So checking for an option is done using the bitwise AND operator. In C this is `&`.

It might be assumed safe to check as follows:

```
- (BOOL)isCrossDisolveUsedInAnimationOptions:(AnimationOptions)options
{
  return (options & TransitionCrossDissolve);
}
```

# But This is Wrong!

Consider the scenario where my animation is created as follows:
```
MyAnimation *animation = [MyAnimation animationWithOptions:TransitionCurlUp];
```

This passes in the `TransitionCurlUp` option (i.e. 48). When this option is passed into the `-isCrossDisolveUsedInAnimationOptions:` method above, it returns `YES`, which is WRONG!

It returns `YES` because the following are equivalent:

* `TransitionCurlUp & TransitionCrossDissolve`
* `48 & 80`
* `0011 0000` bitwise AND `0101 0000`
* `0001 0000`
* non-zero
* `YES`.

It is wrong because my animation options were specified with `TransitionCurlUp` option, but this method says it is a `TransitionCrossDissolve` animation.

# The Correct Way
The correct way to check for this value is to first bitwise AND the values and then check for equality to the required value, as follows:

```
- (BOOL)isCrossDisolveUsedInAnimationOptions:(AnimationOptions)options
{
  return ((options & TransitionCrossDissolve) == TransitionCrossDissolve);
}
```

Or more generally:

```
#define OptionsHasValue(options, value) (((options) & (value)) == (value))
```

With this macro, I can change my method above, as follows:

```
- (BOOL)isCrossDisolveUsedInAnimationOptions:(AnimationOptions)options
{
  return OptionsHasValue(options, TransitionCrossDissolve);
}
```

So go forth and use `NS_ENUM` and `NS_OPTIONS` how they were intended to be used. But just be sure that you are checking their values in the correct way!

[Follow me on twitter](http://twitter.com/dodsios) or subscribe to my [RSS feed](http://octopress.dev/atom.xml) for more of the same!

