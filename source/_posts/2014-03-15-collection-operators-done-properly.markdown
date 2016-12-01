---
layout: post
title: "Collection Operators Done Properly"
date: 2014-07-25 20:33:59 +0000
comments: true
published: true
categories: iOS OSX Objective-C
---

# The Problem

Foundation's [KVC Collection Operators](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/CollectionOperators.html) are often overlooked and underused, but for those in the know they are a powerful tool to have in your Cocoa shed.
The obvious advantage is their consicion, but the big disadvantage is that we don't get compile-time errors when we misuse them and nor do we get code-completion. And herein lies the problem.
In this article I'll discuss KVC Collection Operators in more detail and propose a solution to this problem.

<!-- more -->


# The Desirables

Firstly, we should be explicit about what types of objects we expect to be in the collection. This should give us code-completion on the key path and an error if the key path doesn't exist on objects of the pre-specified type.

Secondly, some collection operators demand that the key path leads to a specific type of object. For example, `@maximum` and `@minimum` require the objects to implement the `-compare:` method. In these cases, we should get a compile-time error if the key path leads to an object of an invalid type.

Finally, we should only be allowed to use the documented collection operators, and we should get compile-time errors if we try to use an undefined operator.


# Some Background

Collection operators have been talked about a lot, but still I see them rarely used. Do people just forget about them? [NSHipster](http://nshipster.com/kvc-collection-operators/) has given an overview of them and Nicolas Bouilleaud has gone into tremendous detail and even [implemented his own](http://bou.io/KVCCustomOperators.html), albeit using an undocumented 'feature' of Cocoa.

I'm not going to repeat what everyone else has written about, and I'm not going to compete with NSHipster :)

As always [the documentation](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/KeyValueCoding/Articles/CollectionOperators.html) is a great place to read up on the topic.


# Proposing A Solution

## **Attempt #1**

I had a few different ideas about how I would like to make this more explicit and achieve what I set out in The Desirables.

The first approach was using a proxy object and a category on `NSArray` like so:

```objc
NSArray *transactions = ...;

// traditional way
NSNumber *total = [transactions valueForKeyPath:@"@sum.amount"];

// my new way:
DZLCollectionProxy *sumProxy = transactions.sum;
NSNumber *total = sumProxy[@"amount"];

// or more simply:
NSNumber *total = transactions.sum[@"amount"];

// and we could get all the payees' distinct accounts like so:
NSArray *payees = transactions.distinctUnionOfObjects[@"payee.account"];
```

This requires `-sum`, `-unionOfObjects` and all other collection operators to be defined in the category on `NSArray`, `NSSet`, etc.

These methods return a proxy object which internally knows which collection operator to perform. The proxy class implements `-objectForKeyedSubscript:`, which allows us to pass the key path in square brackets. See NSHipster's explanation of [Custom Object Subscripting](http://nshipster.com/object-subscripting/).

The nice thing about this is that it is explicit in its use of the collection operator and for that we will get a compile-time error. But the key path can still be undefined.

## **Attempt #2**

I want the key path to be written outside of a string so that we can get code-completion and compile-time errors if the key path is undefined on the objects of a pre-specified type.

Assuming I create my proxy object with the `-each` method, what about if we cast the proxy object to the type of object inside the collection? Then we can specify at the end of the key path which operator to use. Something like so:

```
NSArray *payees = [(id)((Transaction *)transactions.each).payee unionOfObjects];
```

> I convert to `id` here so that the compiler will allow me to send `-unionOfObjects` to a `Payee *`, but I could also declare this method in a category on `NSObject`.

This approach would return a new proxy object for each key in the path, storing the original collection object and keeping a track of the key path. It achieves this by implementing `-resolveInstanceMethod:`, and methods for all collection operators: `-unionOfObjects`, etc.

I like this solution, because it provides code-completion and compile-time errors. But it has a major disadvantage...

Collection operators treat methods (including property getters) that return `NSNumber *` and `double` as if they were the same. For example, if the `Transaction *` class had the following properties:

```
@property (nonatomic, assign) CGFloat amount;
@property (nonatomic, strong) NSNumber *value;
```

I could use collection operators on a group of transactions like so:

```
NSNumber *totalAmount = [transactions valueForKeyPath:@"@sum.amount"];
NSNumber *totalValue = [transactions valueForKeyPath:@"@sum.value"];
```

Now if I want to use my new solution on the `value` object I can do so like this:

```
NSNumber *totalValue = ((Transaction *)transactions.each).value.sum;
```

Looks great right? Simple, explicit, consice. Does what it says on the tin.

But this won't work with the `amount` property. Even though at runtime it would be fine (because the `value` method would be implemented dynamically by my proxy object and return another proxy object), the compiler is never going to let me send the `-sum` message to something of type `CGFloat`.

I can get around this by having another class that does the operations and takes proxy objects as parameters to its methods. For example:

```
NSNumber *totalValue = [DZLCollectionOperator sumNumber:((Transaction *)transactions.each).value];
NSNumber *totalAmount = [DZLCollectionOperator sumDouble:((Transaction *)transactions.each).amount];
```

At runtime, the pointer for the proxy object returned by the `-amount` method will be passed to the `-sumFloat:` method as a `double`. But it can be used by converting it as so, although admittedly it's massively hacky!

```
id proxyObj = (__bridge id)(void *)(long long)aDouble;
```

This solution now also has the advantage of checking at compile-time that the properties are of the correct type for these methods.

But it's a really hacky approach and we still don't get validation on objects that must implement the `-compare:` method.


## **Attempt #3 (Final)**

Macros. It's simple, all we need to do is define some macros that do all the validation and are written in such a way that Xcode will give us code-completion on the key path.

We can use the fact that Xcode is constantly compiling your code as you write it to check for errors. This means that as we use a macro, its already going through the preprocessor as we write it.

My finally proposed solution looks like this:

```
NSNumber *average = DZLAverage(transactions, Transaction *, amount);
NSDate *latestDate = DZLMaximum(transactions, Transaction *, date);
NSArray *accounts = DZLUnionOfObjects(transactions, Transaction *, payee.account);
```

Checks are automatically carried out on `DZLMinimum` and `DZLMaximum`. But I provide extra macros for extra checks for the operators that can process numeric values:

```
DZLSumDouble(self.transactions, Transaction *, amount);
DZLSumNumber(self.transactions, Transaction *, value);

DZLAverageDouble(self.transactions, Transaction *, amount);
DZLAverageNumber(self.transactions, Transaction *, value);
```

Because of the way the macros are written, we get code-completion, nice syntax highlighting in Xcode, warnings and errors as we write the code, and it is simple, explicit and consice.

# Adopt these macros in your project

These macros are [avilable on GitHub here](https://github.com/samdods/DZLCollectionOperators), they come as a single header file, so they're super easy to install in your own project.

You can also install it as [a cocoapod](http://cocoapods.org/?q=dzlcollectionoperators)!

As always, I love discussing things like this in more depth, so [tweet me](http://twitter.com/dodsios), [follow me](http://twitter.com/dodsios) or subscribe to my [RSS feed](http://octopress.dev/atom.xml) for more of the same :)



