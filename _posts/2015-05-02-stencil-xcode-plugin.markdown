---
redirect_from: "/blog/2015/05/02/stencil-xcode-plugin/"
description: Article
layout: post
title: "Introducing Stencil"
date: 2015-05-02 11:50:25 +0100
comments: true
published: true
categories: blog
---

[Stencil](/stencil-xcode-plugin/) is an Xcode plugin that provides the ability to create custom file templates and use them in your project to create new files. It's intended to save you time while making it easy to extend a project in the future: for you, next week; for your current colleagues; and especially for people that might inherit a project from you in the future.

Once you've used [Stencil](/stencil-xcode-plugin/) to create a custom template, you can create new files based on this template. The new files can include Objective-C headers and implementation files, interface builder files (.xib and .storyboard), as well as Swift source files.

It really adds benefit when there's a particular class that you commonly subclass, especially if the new subclass requires an interface builder counterpart.

Of course you would want to document your project to explain that you have defined custom templates, which can be used with the [Stencil](/stencil-xcode-plugin/) plugin. But if one of your co-workers doesn't have the plugin, then it doesn't matter. No harm done, it just means they won't reap the benefits. And you can spend longer on your tea break, while they're spending longer doing things manually!

<!-- More -->

# Creating a custom file template, by example

In a hypothetical social-network-like project, I have an abstract `SectionViewController` class, which will be subclassed for various sections of the app. For example, we have a `FriendsSectionViewController` and in the future we might want to add a `GroupsSectionViewController` and an `EventsSectionViewController`.

The `SectionViewController` interface looks like this:

```objc
@interface SectionViewController : UIViewController
@property (nonatomic, weak) IBOutlet UITableView *tableView;
@property (nonatomic, strong) Section *section;

// may be implemented in subclasses, invoked when a table cell is selected.
- (void)didSelectItemAtIndexPath:(NSIndexPath *)indexPath;
@end
```

It's implementation takes care of the table's data source using the specified `Section` object, and implements the necessary delegate methods.

Subclasses are simply expected to implement the `-didSelectItemAtIndexPath:` method as described in the interface. Each subclass will also have its own storyboard file, which must link up a `UITableView` to the outlet in the superclass and provide a prototype cell with a specific reuse identifier. The `UITableView` will also hook up its `delegate` and `dataSource` outlets to the view controller.

In order to create a new subclass, say `GroupsSectionViewController`, I would need to create the new class files (`.h` and `.m`) and create separately the new storyboard file. In the implementation, I would need to override the `-didSelectItemAtIndexPath:` method. In the storyboard, I would need to add a new view controller, embedded in a navigation controller; add a `UITableView` and hook it up to the outlet; hook up the table's `delegate` and `dataSource` outlets; and add a prototype cell and give it the necessary reuse identifier, as required by the `SectionViewController` implementation. Sounds tedious!

## Simplifying repetitive tasks

This is where [Stencil](/stencil-xcode-plugin/) comes in. We can create a template based on an existing subclass of `SectionViewController`, to simplify adding a new section in the future. This will not only save my time, but make it much easier for a new joiner to get up to speed with the project and get stuck in making a new section, following the pattern that already exists in the project.

Simply select the files that you wish to base your template upon, and select the new option to _New Template from Selection_, as in the image below.

<img src="/assets/images/stencil/create-new-template.jpg" align="center" alt="Create File Template from Group"/>

In the options window that is presented, I'll leave the _Interface or protocol_ option as "interface FriendsSectionViewController", because this is the interface I wish to make a template from (it's the only interface defined in the selected files).

I'll change the _New will inherit from_ option to "SectionViewController" because this is what I want newly created files to inherit from. (I don't want them to inherit from FriendsSectionViewController.)

The _Template name_ is defaulted to "SectionViewController" so I'll leave that, and I'll add a useful description so people know what it is in the future. I'll leave all other options as the defaults. See image below.

<img src="/assets/images/stencil/create-template-options.jpg" align="center" alt="Template Creation Options"/>

You will then be presented with the template files for you to edit. Ensure you don't modify the parts of the code that have references like `___FILEBASENAMEASIDENTIFIER___`, as this is what Xcode will replace when you use your template in the future.

Remove any functionality that is specific to the class you used to create the template. Leave stubs of methods that you expect to be implemented in new classes that you create based on your template. See image below.

<img src="/assets/images/stencil/editing-new-template.jpg" align="center" alt="Editing The Template"/>

If your template includes an Interface Builder file, the template file will be presented in a separate window. Remove any views or controllers that are specific to the class you used to create the template. You can leave outlets connected and don't modify the custom class `___FILEBASENAMEASIDENTIFIER___`. See image below.

<img src="/assets/images/stencil/editing-ui-template.jpg" align="center" alt="Editing Interface Builder Template"/>

After following the instructions that are presented to you in an Xcode window, make your changes to the template files, close the window(s) and return to your project.

That's it! You've now created your own custom file template. It will be saved under your project's directory (under the `StencilPlugin` subdirectory), so you can commit it to source control.

# Using your template to create a new file

So far in my project, I only have a `FriendsSectionViewController`. Now it's time to add a `GroupsSectionViewController`. To use the template created above, simply right-click in the Project Navigator and select _New File from Custom Template..._. This will present all templates that are associated with this project. See image below.

<img src="/images/stencil/new-from-template.jpg" align="center" alt="New File from Custom Template"/>

Select the `SectionViewController` template created previously. See image below.

<img src="/images/stencil/select-custom-template.jpg" align="center" alt="Selecting Custom Template"/>

Give your new class a name, in my case this will be `GroupsSectionViewController`. Your new files will be created, just like usual. You'll see the code files and the Interface Builder files all present in the Project Navigator. The storyboard file has the `UITableView` hooked up, as well as its `delegate` and `dataSource`. The code files have stubs of the methods that I'm expected to implement in the new subclass. See image below.

<img src="/images/stencil/new-files-created.jpg" align="center" alt="New Files Created"/>

# Conclusion

[Stencil](/stencil-xcode-plugin/) may not provide benefit to every kind of project. But generally -- if extending the app means that you frequently subclass a particular class -- then you can find benefits in using the [Stencil](/stencil-xcode-plugin/) plugin. You might find yourself subclassing a `WidgetView` for example, it's much quicker to use [Stencil](/stencil-xcode-plugin/) to produce your new files, which may include a `.xib` file. And it makes it immediately clear to people joining the project in the future what methods are expected to be implemented in new subclasses.

# Get Stencil

See the [Stencil](/stencil-xcode-plugin/) website.
