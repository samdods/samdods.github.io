---
layout: post
title: "Dynamic Accessors for Category Properties"
date: 2014-01-04 10:06:30 +0000
comments: true
author: Sam Dods
categories: Objective-C-Runtime
---
# The Problem
Repetition, repetition, repetition. I don't like it. I like code to be as <a href="http://en.wikipedia.org/wiki/Don't_repeat_yourself">dry</a> as it can possibly be without affecting readability, so as soon as I find myself repeating code in a project I immediately refactor. And if I find myself repeatedly writing what many would consider boilerplate code, I begin to wonder how I can avoid doing this in the future.

I was recently adding a couple of properties to an existing class using a category. We can do this thanks to the Objective-C runtime functions `objc_getAssociatedObject` and `objc_setAssociatedObject`. I found myself writing the same old getters and setters that looked identical to those I'd written countless times before. And herein lies the problem. I don't like repeating myself.
<!-- more -->

# A Solution
Below is an example of an Objective-C category with some very important properties. It's a surprise these don't exist on the underlying class.
``` objc
@interface BondGirl (DZLAdditions)
@property (strong, nonatomic) UIColor *dzl_hairColor;
@property (strong, nonatomic) NSArray *dzl_weapons;
@end
```
The nicest solution I came up with was to simply declare the properties as dynamic and implement the necessary accessor methods at runtime, from the `+load` method.
```
@implementation BondGirl (DZLAdditions)
@dynamic dzl_hairColor, dzl_weapons;

+ (void)load
{
  // implement for all properties (where methods are not already defined)
  [self implementDynamicPropertyAccessors];
  // or implement only for specific property specified by name
  [self implementDynamicPropertyAccessorsForPropertyName:@"dzl_weapons"];
  // or implement only for specific properties match regular expression
  [self implementDynamicPropertyAccessorsForPropertyMatching:@"^dzl_"];
}

@end
```

I think it looks nice. And if you have multiple properties in your category then it can save a lot of lines of code, and ultimately it avoids repetition. The documentation tells us that a category `+load` method is called after the class's own `+load` method. The important thing here is that implementing the `+load` method within a category does not override the `+load` method of the class itself.

I've found that the `+implementDynamicPropertyAccessors` method is always sufficient for my needs. But the other methods may provide peace of mind as well as the ability to use this functionality on a class that has other dynamic properties that you don't wish to be implemented in this way.

# The Working
All the magic happens in the `+[NSObject implementDynamicPropertyAccessors]` method defined in a category on NSObject.

The method itself is very simple.
```
+ (void)implementDynamicPropertyAccessors
{
  [self enumeratePropertiesWithBlock:^(objc_property_t property){
    [self implementAccessorsIfNecessaryForProperty:property];
  }];
}
```
All I'm doing here is iterating over the properties of `self`, and sending each property off to another method, which will do all the hard work.

The method `+implementAccessorsIfNecessaryForProperty:` will inspect the attributes of the property to determine whether or not the accessors need to be implemented. It will then implement the accessor methods if necessary.

```
+ (void)implementAccessorsIfNecessaryForProperty:(objc_property_t)property
{
  // 1.
  NSArray *attributes = [self attributesOfProperty:property];
  BOOL isDynamic = [attributes containsObject:@"D"];
  if (!isDynamic) {
    return;
  }

  // 2.
  BOOL isObjectType = YES;
  NSString *customGetterName;
  NSString *customSetterName;

  for (NSString *attribute in attributes) {
    unichar firstChar = [attribute characterAtIndex:0];
    switch (firstChar) {
      case 'T': isObjectType = [attribute characterAtIndex:1] == '@'; break;
      case 'G': customGetterName = [attribute substringFromIndex:1]; break;
      case 'S': customSetterName = [attribute substringFromIndex:1]; break;
      default: break;
    }
  }
  if (!isObjectType) {
    return;
  }

  // 3.
  static const void *key = &key;
  key++;

  // 4.
  const char *name = property_getName(property);
  [self implementGetterIfNecessaryForPropertyName:name customGetterName:customGetterName key:key];

  BOOL isReadonly = [attributes containsObject:@"R"];
  if (!isReadonly) {
    [self implementSetterIfNecessaryForPropertyName:name customSetterName:customSetterName key:key attributes:attributes];
  }
}
```
1. First I fetch the attributes of the property and check if the property is dynamic. If the property is not dynamic then we have no business with it.
2. Next I go through the attributes to determine the type of the property and the custom getter and setter names, if specified. If the property is not an object type, then we have no business with it (I don't currently support properties that are not of an object type).
3. Next I create a key to be used as a reference to the associated object. Note: this is a static variable, which I increase for each property being implemented, allowing for 2^32 properties to be implemented on each object (or 2^64 for a 64-bit binary), which I think is more than sufficient.
4. And finally, I get the name of the property and implement the getter and setter methods. The setter is only implemented if the property is not marked as readonly. In reality the property will very rarely be readonly, because something has to set the value. I occasionally define a readonly property in a category and set it lazily when it's read. I might do this, for example, with an NSMutableArray property, but I'm not supporting that kind of functionality here.

# The Missing Pieces
Among the missing pieces are the methods that actually implement the getter and setter.
```
+ (void)implementGetterIfNecessaryForPropertyName:(char const *)propertyName customGetterName:(NSString *)customGetterName key:(const void *)key
{
  SEL getter = NSSelectorFromString(customGetterName ?: [NSString stringWithFormat:@"%s", propertyName]);
  [self implementMethodIfNecessaryForSelector:getter parameterTypes:NULL block:^id(id _self) {
    return objc_getAssociatedObject(self, key);
  }];
}
```
The method above defines the getter using the property name or a custom getter name if specified. It then calls another method to create the implementation based on the block specified. The block returns the associated object using the key as the reference.
```
+ (void)implementSetterIfNecessaryForPropertyName:(char const *)propertyName customSetterName:(NSString *)customSetterName key:(const void *)key attributes:(NSArray *)attributes
{
  BOOL isCopy = [attributes containsObject:@"C"];
  BOOL isRetain = [attributes containsObject:@"&"];
  objc_AssociationPolicy associationPolicy = isCopy ? OBJC_ASSOCIATION_COPY : isRetain ? OBJC_ASSOCIATION_RETAIN : OBJC_ASSOCIATION_ASSIGN;
  BOOL isNonatomic = [attributes containsObject:@"N"];
  if (isNonatomic) {
    objc_AssociationPolicy nonatomic = OBJC_ASSOCIATION_COPY_NONATOMIC - OBJC_ASSOCIATION_COPY;
    associationPolicy += nonatomic;
  }

  SEL setter = NSSelectorFromString(customSetterName ?: [NSString stringWithFormat:@"set%c%s:", toupper(*propertyName), propertyName + 1]);
  [self implementMethodIfNecessaryForSelector:setter parameterTypes:"@" block:^(id _self, id var) {
    objc_setAssociatedObject(self, key, var, associationPolicy);
  }];
}
```
The method above defines the setter using the property name -- capitalized and prefixed with `set` -- or a custom setter name if specified. It also has to inspect the attributes to determine the association policy, which must be provided when setting an associated object. The block used here simply sets the associated object using the specified key as the reference.

```
+ (void)implementMethodIfNecessaryForSelector:(SEL)selector parameterTypes:(const char *)types block:(id)block
{
  BOOL instancesRespondToSelector = [self instancesRespondToSelector:selector];
  if (!instancesRespondToSelector) {
    IMP implementation = imp_implementationWithBlock(block);
    class_addMethod(self, selector, implementation, types);
  }
}
```
The method above adds a method to the class for the given selector and implementation block.

Finally, I defined a couple of other methods to separate out concerns. The following method iterates over the properties and executes the given block with each property
```objc
+ (void)enumeratePropertiesWithBlock:(void(^)(objc_property_t property))block
{
  NSParameterAssert(block);
  uint count = 0;
  objc_property_t *properties = class_copyPropertyList(self, &count);
  for (uint i = 0; i < count; i++) {
    objc_property_t property = properties[i];
    block(property);
  }
  free(properties);
}
```

And the following method simply returns an array of the attributes of the given property.
```objc
+ (NSArray *)attributesOfProperty:(objc_property_t)property
{
  return [[NSString stringWithCString:property_getAttributes(property) encoding:NSUTF8StringEncoding] componentsSeparatedByString:@","];
}
```
<br>
Thanks for reading! If you got this far, I hope it's because you found it interesting. If I'm right, then you might like to check out the code, which is available on <a href="https://github.com/samdods/dynamicCategoryProperties/tree/master/DynamicCategoryProperties/NSObject%2BCategoryProperties">GitHub</a>. It's part of a project to demo how it can be used, and is tweaked to include extra methods `+implementDynamicPropertyAccessorsForPropertyName:` and `+implementDynamicPropertyAccessorsForPropertyMatching:`.

And [follow me on twitter](http://twitter.com/dodsios) or subscribe to my [RSS feed](http://octopress.dev/atom.xml) for more of the same!



