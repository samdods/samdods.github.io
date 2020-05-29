---
redirect_from: "/blog/2014/04/08/user-defined-runtime-attributes-in-interface-builder/"
description: Article
layout: post
title: "Pushing the Limits of User-Defined Runtime Attributes in Interface Builder"
date: 2014-04-08 19:16:47 +0100
comments: true
categories: blog
---

User-defined runtimes attributes in Xcode's Interface Builder are a great way to keep view controller and view code clean, while obeying Separation of Concerns.
They allow you to configure properties on the view or view controller that you are unable to configure from within Interface Builder's *Attributes Inspector* or *Size Inspector*.

But sometimes you may want to setup from Interface Builder a scenario which you can't do from simple KVC-compliant property manipulation.

One example I ran into recently was trying to set the content insets on a collection view in a normal view controller in an iPhone-specific storyboard. In Xcode 5.1 you are unable to set the content insets of a collection view (I'm not sure in which Xcode version this was first missing - I'm sure you used to be able to do it).
So with this feature missinh, it would be nice to be able to do the same thing using user-defined runtime attributes.

Because contentInset is of `UIEdgeInsets` type, this isn't possible with any of the types defined in Interface Builder under user-defined runtime attributes. But it can be done!

Another example I will show below is how to set the borderColor property of a `CALayer`, even though the runtime attributes don't support `CGColorRef` type. All will be explained!

<!-- More -->

# Runtime Attributes Explained

First a bit of background: user-defined runtime attributes can be set from the *Identity Inspector* tab in Interface Builder utilities, as shown below:

<img src="/assets/images/user-defined-runtime-attributes.png" align="center" alt="User-defined runtime attributes in Xcode's Interface Builder"/>

They are defined as a set of key-value pairs. The key is actually a *key path*, which is a powerful, especially in Interface Builder where you don't have direct access to the underlying `CALayer` of a `UIView` (more on this below).

The limitation is that the object's class must be KVC-compliant for the key (or key path) defined in the set of runtime attributes. To be *KVC-compliant* for a key `foo` simply means instances of the class must respond to the selector `setFoo:`.
> Really, being KVC-compliant for a property means the class should implement the methods required for `valueForKey:` and `setValue:forKey:` to work for that property. For user-defined runtime attributes, that means implementing the setter for the property (key).

The following property types available for runtime attributes in Interface Builder:

* **Boolean** - translates to a `BOOL` property
* **Number** - can translate to any numeric scalar property or a property of type `NSNumber *`
* **String** - translates to a property of type `NSString *`
* **Localized String** - the value here is a key to look up in the `strings` file for the current locale.
* **Point** - translates to a `CGPoint` property
* **Size** - translates to a `CGSize` property
* **Rect** - translates to a `CGRect` property
* **Range** - translates to an `NSRange` property
* **Color** - translates to a property of type `UIColor *`
* **Nil** - this spectial type doesn't allow you to set a value, it is just a way of specifying that the value should be set to `nil`

> The types **Point** and **Size** can be used interchangeably, because they both map to the same type of C structure (`struct {CGFloat v1; CGFloat v2;}`).


# Some Examples


### Configuring a CALayer

A great example of using runtime attributes is to configure a `UIView`'s underlying `CALayer`. For example, we can set the layer's border width and corner radius as follows:

<img src="/assets/images/calayer-setup.png" align="center" alt="Configuring a view's underlying layer"/>

Unfortunately, Interface Builder doesn't allow us to set the color of a CALayer, because the **Color** type doesn't translate to properties of `CGColorRef *` type (workaround discussed below).

### Configuring Custom Controls

Another example is if you are using a custom `UIControl` object such as a range slider. A range slider is similar to the built-in slider, but has two thumbs, or knobs: one to specify the minimum value and one to specify the maximum value.
This kind of control would be useful for setting a minimum price and maximum price in a search query. There are [various](https://github.com/muZZkat/NMRangeSlider) [implementations](http://www.sitepoint.com/wicked-ios-range-slider-part-one/) [available](https://github.com/barrettj/BJRangeSliderWithProgress) in the community.

Using user-defined runtime attributes, you can configure such a control right from within Interface Builder. Taking the example of [NMRangeSlider](https://github.com/muZZkat/NMRangeSlider), you could configure the minimum and maximum values for the slider as follows:

<img src="/assets/images/custom-control-setup.png" align="center" alt="Configuring a custom control"/>

The main benefit of Interface Builder in my opinion is that it keeps all the UI configuration logic in one place outside of the view controller. By configuring your controls in Interface Builder, this is yet more code that can be removed from your view controller. After all, you would configure your `UIButton` and `UISlider` controls in Interface Builder, so why not configure your custom controls too?


# The Limitations

As discussed above, the class must be KVC-compliant for each key you specify in the runtime attributes. If you specify a key path, then the object returned for each key must be KVC-compliant for the following key in the path.

For example, the list of types shown above means that we can't set the content inset on a `UICollectionView` because there is no `UIEdgeInset` type (and no type that uses the exact same C structure type).

We also already discussed the fact that we can't set the border color of a `CALayer`. Wouldn't it be nice if anything you can think of, you could configure in Interface Builder without adding logic to your view or view controller?


# The Workaround

Define a category on the object you wish to configure. Comply to KVC in the category and put the configuration logic in the setter method. You don't even need to `#import` the category header anywhere - just by having it in the project, the runtime will call the setter method.

### Define a Category

For example, by defining a category on `UICollectionView`, we can make the class KVC-compliant for any key we choose. To set the content inset, we can define the following category implementation:

```objc
@implementation UICollectionView (ContentInset)

- (void)setContentInsetFromString:(NSString *)string
{
  self.contentInset = UIEdgeInsetsFromString(string);
}

@end
```

There's no need to define anything in the category header.

This particular example is especailly simple because of the `UIEdgeInsetsFromString` mathod, which could it seems have been created specifically for this purpose!

### Configure the Interface

Now we can set the user-defined runtime attributes in Interface Builder like so:

<img src="/assets/images/content-inset-from-string.png" align="center" alt="Configuring content inset"/>

The **Key Path** must match the name of the setter method in the category, but without the **set** prefix and with lowercase first letter.

Now you've configured your interface in Interface Builder - the correct place - and you can avoid having clutter in your code!

An added bonus of doing things in Interface Builder is that if you use separate nibs or storyboards for iPad/iPhone then you can configure your view differently for each device without cluttering your code checking the user interface idiom.


# Another Example

Now we can define the border color of a `CALayer` by simply mapping it from a `UIColor`.

### Define a Category

```
@implementation CALayer (Additions)

- (void)setBorderColorFromUIColor:(UIColor *)color
{
  self.borderColor = color.CGColor;
}

@end
```

### Configure the Interface

<img src="/assets/images/calayer-border-color.png" align="center" alt="Configuring a layer's border color"/>

# Considerations

I think this is the right way to configure a user interface. Given that we have Interface Builder, we should use it to its full potential. But... there's always a "but" isn't there?

Many people wouldn't think to look in Interface Builder at the user-defined runtime attributes, so they could be left very puzzled if they come to maintain your code at a later date. They could be left with a seemingly random border to their view, unable to figure out why it's there.

So like anything of this nature -- and by that I mean a more-advanced use of a programming language or development evironment -- we should use it with caution. Use it only if it is necessary and simplifies logic elsewhere in your code.

Setting up a view in a view controller isn't necessarily messy. The most valid reason for using user-defined runtime attributes is to configure a view differently whether the Interface Builder file is for iPhone or iPad. Because you already have a separate nib or storyboard file specifically for each device, that should be the place that you do any device-specific configuration.


I hope you enjoyed reading this article. If so, please [follow me on twitter](https://twitter.com/dodsios) or subscribe to my [RSS feed](http://octopress.dev/atom.xml) for more of the same.



