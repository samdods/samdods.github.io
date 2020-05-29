---
redirect_from: "/blog/2014/08/03/secondary-delegate/"
description: Article
layout: post
title: "How to Intercept Delegate Messages If You Are Not The Delegate"
date: 2014-08-03 13:45:25 +0100
comments: true
published: true
categories: blog
---

I recently found myself in a situation where I needed to listen out for changes to a UIScrollView of which I couldn’t set the delegate. My situation was with the underlying UIScrollView of a UIWebView instance, but there are other situations where you might want to receive delegate method calls from an object for which you are not allowed to set the delegate.

The problem is that UIWebView exposes its underlying UIScrollView, but it would be bad etiquette to set its delegate, because you are not the owner of the UIScrollView instance and you don’t know the implementation of UIWebView.

So, the question is: how can we listen out for the delegate messages from a UIScrollView, when we are not acting as the delegate?

<!-- More -->

One solution would be to use KVO. But it wouldn't be possible to implement `-scrollViewWillEndDragging:withVelocity:targetContentOffset:` using KVO.

# Secondary Delegate

So I propose adding a `secondaryDelegate` property in a category on UIScrollView. This category will implement the `-setSecondaryDelegate:` method, which can be called from the app (from a view controller, say).

```objc
@interface UIScrollView (SecondaryDelegate)

@property (nonatomic, weak) id<UIScrollViewDelegate> secondaryDelegate;

@end
```

So for example, my view controller can set itself as the “delegate” of its web view’s UIScrollView as follows:

```
- (void)viewDidLoad
{
  [super viewDidLoad];

  self.webView.scrollView.secondaryDelegate = self;
}
```

By setting the `secondaryDelegate` property, the category’s setter method actually overrides the `delegate` property with a `delegateProxy` object. This proxy object holds a weak reference to the original (overridden) delegate and to the secondary delegate provided.

# Delegate Proxy

The delegate proxy is responsible for forwarding messages to the relevant delegates. The proxy holds a weak reference to each delegate. This means it is perfectly acceptable for the UIScrollView to hold a strong reference to the proxy by way of associated object (explained in more detail later).

The proxy object is a subclass of NSProxy and conforms to the UIScrollViewDelegate protocol, so it must implement the following methods: `-methodSignatureForSelector:` and `-forwardInvocation:`.

We’ll also implement the `-respondsToSelector:` method, because we know our proxy will be used as the delegate of a UIScrollView, which will check if the delegate responds to each selector before calling that selector.

To determine the method signature for a selector, we can just ask each delegate and return when we find one that can provide what we need.

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector
{
  NSObject *delegateForResonse = [self.primaryDelegate respondsToSelector:selector] ? self.primaryDelegate : self.secondaryDelegate;
  return [delegateForResonse respondsToSelector:selector] ? [delegateForResonse methodSignatureForSelector:selector] : nil;
}
```

To determine if the proxy can respond to a selector, again we just check each delegate and return `YES` if one of them can respond.

Before calling one of its optional delegate methods the UIScrollView (or anyone else for that matter) asks its delegate if it can respond to the selector. If it replies with `YES`, then it will send the message. But because we have set the `delegate` property to our `delegateProxy` object, this message will arrive as an invocation in the `-forwardInvocation:` method.

We should implement this method as follows in order to forward the invocation to each delegate in turn (if it can respond to the selector).

```
- (void)forwardInvocation:(NSInvocation *)invocation
{
  [self invokeInvocation:invocation onDelegate:self.primaryDelegate];
  [self invokeInvocation:invocation onDelegate:self.secondaryDelegate];
}

- (void)invokeInvocation:(NSInvocation *)invocation onDelegate:(id<UIScrollViewDelegate>)delegate
{
  if ([delegate respondsToSelector:invocation.selector]) {
    [invocation invokeWithTarget:delegate];
  }
}
```

# The UIScrollView Category

For completeness, I’ve added the code for the category, with comments below.

```
// 1. Private interface extension
@interface UIScrollView ()
@property (nonatomic, strong) TABScrollViewDelegateProxy *delegateProxy;
@end

@implementation UIScrollView (SecondaryDelegate)

// 2. Setter
- (void)setSecondaryDelegate:(id<UIScrollViewDelegate>)secondaryDelegate
{
  if (!self.delegateProxy) {
    self.delegateProxy = [TABScrollViewDelegateProxy alloc];
    self.delegateProxy.primaryDelegate = self.delegate;
  }
  
  self.delegateProxy.secondaryDelegate = secondaryDelegate;
  self.delegate = self.delegateProxy;
}

// 3. Getter
- (id<UIScrollViewDelegate>)secondaryDelegate
{
  return self.delegateProxy.secondaryDelegate;
}

// 4. Associated object
- (void)setDelegateProxy:(TABScrollViewDelegateProxy *)delegateProxy
{
  objc_setAssociatedObject(self, @selector(delegateProxy), delegateProxy, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (TABScrollViewDelegateProxy *)delegateProxy
{
  return objc_getAssociatedObject(self, @selector(delegateProxy));
}
```

1. The `delegateProxy` property is added to the private interface extension. The fact that this category uses a proxy is an implementation detail and should therefore not be exposed in the public interface.
2. The setter method for the `secondaryDelegate` property will create the `delegateProxy` object if it doesn’t already exist and assign it to the `delegateProxy` property. Next we the `primaryDelegate` to the original delegate held in `self.delegate`. Then we set the `delegateProxy.secondaryDelegate` and override the object held in `self.delegate`. (Notice how we only call `alloc` on the proxy class - this is because instances of NSProxy do not by default respond to `-init`.)
3. We should provide the getter method for the object. It is just a wrapper around getting it from `self.delegateProxy`.
4. We hold a strong reference to the `delegateProxy` object by using an associated object. The strong reference is fine, because it only holds weak references to each delegate. (Notice the use of `@selector(delegateProxy)` as the key for the associated object. This is fine because the pointer returned is unique.)

# Conclusion

This solution is clean and robust. It doesn’t involve method swizzling - only message forwarding using a proxy object, which is fine. In fact the only reason we have to import `<objc/runtime.h>` is to use associated objects.

The public interface is clean - it only provides a single extra property for UIScrollView - the `secondaryDelegate`.

The only downside to this is that the original delegate must have been set (if it ever will be set) before you set the secondary delegate. Or rather, you cannot set the `delegate` property after setting the `secondaryDelegate`. This is because setting the `secondaryDelegate` overwrites the value of the `delegate` property. I thought about a solution that adds an observer to listen for changes to the `delegate` property, but removing the observer on deallocation was not trivial, so I settled on the solution described above, which will not allow the delegate to be set after having set the secondary delegate.

If you found this interesting, please [follow me on twitter](http://twitter.com/dodsios), or [subscribe to my RSS feed](http://samdods.github.io/atom.xml).



