---
layout: post
title: "swift alternatives to c macros (part ii)"
date: 2015-04-07 19:29:07 +0100
comments: true
published: false
categories: iOS OSX Swift
---
In my [previous post](/blog/2015/04/06/swift-alternatives-to-c-macros), I investigated a Swift alternative to one of the more complex types of macro in Objective-C. I didn't actually give an example of the macro definition, only how it was used. Please forgive my insolence.

In this post, I'll briefly recap on my last post for completeness before moving onto show how other examples of Objective-C macros and their alternatives in Swift.

<!-- More -->

# Other cool macros

Some of the coolest macros out there, which are increasingly more widely used, are the `@weakify`/`@strongify` macros provided by [libextobjc](https://github.com/jspahrsummers/libextobjc).

Using these macros allow you to safely use a weak reference in a block that would otherwise retain the reference. For example,

```objc
@weakify(self);
[self.otherObject doSomething:^{
  @strongify(self);
  [self doAnotherThing]; // self can be referenced within this block, but it is not retained!
}];
```

So what's the alternative to this type of macro in Swift?

Well, it's as though Apple developed Swift with this in mind. You don't need any of this funky macro stuff anymore, you simply declare the reference type in the block, like so:

```swift
self.otherObject.doSomething {
  [weak self] in
  self.doAnotherThing()
}
```

Isn't that simple? So for this example of a macro in Objective-C, the alternative is baked right into Swift's syntax.

# Recapping on last time

This is how the KVC-fixer macro could be defined in Objective-C:

```objc
#define DZLMinimum(collection, typesOfObjectInCollection, keyPath) ({ \
if (NO) { ^id(){((typesOfObjectInCollection)(nil)).keyPath; return collection;}(); } \ // this line is optimised out by the compiler
[(collection) valueForKeyPath:@"@min."#keyPath]; })
```

The following shows how it will be expanded by the preprocessor:

```objc
double minimumAccountBalance = DZLMinimum(transactions, Transaction *, account.balance);

/// expands to:

double minimumAccountBalance = ({
  if (NO) {
    ^id(){
      ((Transaction *)(nil)).account.balance;
      return transactions;
    }();
  }
  [transactions valueForKeyPath:@"@min.account.balance"];
})
```

Any code contained in `if (NO) { ... }` will be validated by the preprocessor, but will never actaully be compiled into the binary. It is optimised out by the compiler. So after your friend the compiler has finished with it, it is effectively the same as having written:

```objc
double minimumAccountBalance = [transactions valueForKeyPath:@"@min.account.balance"];
```

The Swift equivalent that I touted in [my previous post](/blog/2015/04/06/swift-alternatives-to-c-macros) is the following:

```swift
var minimumAccountBalance = transactions.mapMinimum {$0.account.balance}
```

And this provides all of the same benefits as the Objective-C macro.

# Moving on Swiftly... (Sorry!)

I was looking for other examples of where I've used macros in Objective-C and stumbled upon this little bad boy:

```objc
#define MapFormatToLabel(format, label) self.label.text = [self evaluateFormatString:cellTemplate.format fieldsByID:fieldsByID]

MapFormatToLabel(primaryHeaderFormat, mainHeaderLabel);
MapFormatToLabel(primaryValueFormat, mainValueLabel);
// repeated multiple times for different mappings...
```

This macro is defined in-line (rather than at the top of the file or in a header file). It simplifies what would otherwise be:

```objc
self.mainHeaderLabel.text = [self evaluateFormatString:cellTemplate.primaryHeaderFormat fieldsByID:fieldsByID];
self.mainValueLabel.text = [self evaluateFormatString:cellTemplate.primaryValueFormat fieldsByID:fieldsByID];
```

Consider the line above being repeated for a dozen different format-to-label mappings.

> It's real code from a complex system whereby the API can specify formats for labels in a widget. The format strings can contain hashtag references to fields, defined by the `fieldsByID` dictionary. The format of each field is defined by the `cellTemplate` object (for example, in the `primaryHeaderFormat` property). The `-evaluateFormatString:fieldsByID:` method expands the format string replacing each hashtag reference with the value from the fields with the corresponding ID.

Anyway, the macro here makes it easier to read, more [DRY](http://en.wikipedia.org/wiki/Don%27t_repeat_yourself), easier to modify and less prone to copy-paste mistakes.

The macro itself is very simple: it takes the property name of the format; gets the value of that property from the `cellTemplate` instance; sends that along with the `fieldsByID` mapping into the method; and uses the result to set the text on a `UILabel` property of `self`, which is also given in the macro "invocation".

# So how can the same be achieved in Swift?

Let's first think about what we are trying to solve.

