---
layout: post
title: "Modal View In Front of the Status Bar"
date: 2014-02-10 07:55:28 +0000
comments: true
categories: iOS Objective-C
---

This post is based on a really cool effect I saw in the Facebook Paper app released last week. In this app, the user can drag a photo up to the top of the screen to view it full-screen. In doing this, the photo actually appears to be in front of the status bar.

<!-- more -->

**Edit** Thanks to [@EricHoracek](https://twitter.com/EricHoracek) for pointing out that this can be done without the use of private API methods. I had originally been adding the top (in-front) view controller's view to the `UIStatusBarWindow` which contains the status bar. But the solution below is much cleaner and uses only public API methods, so is guaranteed for App Store approval.

It's nice to discuss things like this with the never-ending endeavour of finding cleaner approaches and learning something along the way!

# The Effect

<img src="https://github.com/samdods/StatusBarDemo/blob/master/statusBarDemo.gif?raw=true" title="made at imgflip.com"/>

What I really like about this is that you can display content full-screen, with no status bar, but allow the user to simply drag down slightly to see the time and other activity shown in the status bar. As we all know with the arrival of iOS 7, content is king! And I think it looks great to show images and other full-screen content without the status bar. But it's nice to know that it's only a slight drag away.

# The Secret

It's actually very easy. `UIWindow` has the `windowLevel` property, which is of type `UIWindowLevel` (i.e. `CGFloat`). There are a few predefined window levels that we can take advantage of for this trick. The predefined values are as follows:

```objc
UIKIT_EXTERN const UIWindowLevel UIWindowLevelNormal;
UIKIT_EXTERN const UIWindowLevel UIWindowLevelAlert;
UIKIT_EXTERN const UIWindowLevel UIWindowLevelStatusBar;
```

When displaying a new window, it appears in front of all other existing windows at the same level. The higher the `windowLevel` value, the closer to the front of the screen the window appears.

Below is how to create a new window and add it above the status bar.

```objc
// 1.
self.topWindow = [[UIWindow alloc] initWithFrame:self.view.bounds];

// 2.
[self.topWindow setRootViewController:self.overlayViewController];

// 3.
self.topWindow.windowLevel = UIWindowLevelStatusBar;

// 4.
[self.topWindow makeKeyAndVisible];
```

1. Create a new window. Although `UIWindow` is a subclass of `UIView`, instances of this class don't generally have a superview. You don't need to add the window to anything else in order for it to be displayed.
2. Specify the root view controller for the window. This is the view controller who's view you want to appear in front of the status bar.
3. Specify the window level. By specifying `UIWindowLevelStatusBar`, the new window will appear above the status bar. Other windows with a higher `windowLevel`, for example `UIWindowLevelAlert` will appear in front of your new window still.
4. Show the window. By invoking `makeKeyAndVisible`, the window is added to the application. You could also write `self.topWindow.hidden = NO;`.

# The Demo

I created a demo project [available on GitHub](https://github.com/samdods/StatusBarDemo).

I'd be happy to talk about this, so feel free to [tweet me](http://twitter.com/dodsios). I'd love to hear if anyone else has tried this and maybe taken a different approach. Subscribe to my [RSS feed](http://octopress.dev/atom.xml) for more of the same!
