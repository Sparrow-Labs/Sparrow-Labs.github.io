---
layout: post
title: Some private Goodness
author: oliverletterer
inactive: true
---

{{ page.title }}
================

Inspired by [@steipetes](https://twitter.com/steipete) awesome UIKonf talk [How to bend UIKit to your will](http://www.uikonf.com/speakers/peter_steinberger.html), in which he discusses when and where to use private APIs, I decided to open source one of my hacks where we are using private APIs.

## Introducing section locations
Imaging the following scenario: You have this great design for a custom grouped UITableView which you want to implement. That means you would need to know the location in each section in your UITableViewCell subclass. Since there are no public APIs for this, a first approach for this might be to introduce for own custom section location type

```
typedef enum {
    SLUITableViewCellSectionLocationTop = 2,
    SLUITableViewCellSectionLocationCenter = 1,
    SLUITableViewCellSectionLocationBottom = 3,
    SLUITableViewCellSectionLocationSingle = 4
} SLUITableViewCellSectionLocation;
```

and calculate and apply it in

```
- (UITableViewCell *)cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
	// grab a cell
	UITableViewCellSubclass *cell = ...;
	
	// calculate for own section location
	SLUITableViewCellSectionLocation sectionLocation = ...;
	
	// apply your own calculated section location
	cell.customSectionLocation = sectionLocation;
	
	return cell;
}
```

## Where things get ugly

This all works out pretty well as long as your UITableView content is static and doesn't change. In case you insert or delete any rows at runtime, you will need to reload all surrounding cells where the section location changes. This might get messy but is still doable. Now let's say you would want to support reordering cells as well by implementing `-[UITableViewDataSource tableView:canMoveRowAtIndexPath:]`. While the user is dragging a cell around the screen, the section location changes. The UITableView isn't giving you any callbacks or ways to know when these changes occure and you end up with a bad UX. If there was just an Apple provided API for this ...

## The private Goodness: `-[UITableViewCell setSectionLocation:animated:]`

It turns out that Apple ships and supports the following private API

```
@interface UITableViewCell

- (SLUITableViewCellSectionLocation)sectionLocation;
- (void)setSectionLocation:(SLUITableViewCellSectionLocation)sectionLocation;
- (void)setSectionLocation:(SLUITableViewCellSectionLocation)sectionLocation animated:(BOOL)animated;

@end
```

since at least [__iOS 2.1__](https://github.com/nst/iOS-Runtime-Headers/blob/2.1/Frameworks/UIKit.framework/UITableViewCell.h) up until __iOS 6.1__ of this writing. There are a few [Radars out there](http://openradar.appspot.com/11829507) demanding these APIs to be public. Since Apple is supporting these APIs for such a long time, it is likely for Apple to make them public at some point and don't suddenly remove them. This might possibly happen with iOS 7 or not. Until that happens, we decided to make these API available for any UITableViewCell subclass without worrying about AppStore rejection.

## Hacking at runtime

Implementing these methods at compile time will most likely get you rejected by Apple. So we decided to expose this _new_ public interface:

```
@interface UITableViewCell (SLUITableViewCellSectionLocation)

@property (nonatomic, assign) SLUITableViewCellSectionLocation forbiddenSectionLocation;
- (void)setForbiddenSectionLocation:(SLUITableViewCellSectionLocation)location animated:(BOOL)animated;

@end
```

which you can implement at runtime and call super on. At runtime, `UITableViewCell (SLUITableViewCellSectionLocation)` adds `setSectionLocation:` and `setSectionLocation:animated:` to your UITableViewCell subclass

```
- (id)__SLUITableViewCellSectionLocationInitWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier __attribute__((objc_method_family(init)))
{
    if ((self = [self __SLUITableViewCellSectionLocationInitWithStyle:style reuseIdentifier:reuseIdentifier]) && self.class != [UITableViewCell class]) {
        SEL selector = NSSelectorFromString([@[@"set", @"Section", @"Location:"] componentsJoinedByString:@""]);
        if ([self.class instanceMethodForSelector:selector] == [UITableViewCell instanceMethodForSelector:selector]) {
            IMP implementation = imp_implementationWithBlock(^(UITableViewCell *_self, SLUITableViewCellSectionLocation location) {
                [_self setForbiddenSectionLocation:location];
            });
            class_addMethod(self.class, selector, implementation, method_getTypeEncoding(class_getInstanceMethod(self.class, @selector(setForbiddenSectionLocation:))));
        }
        
        selector = NSSelectorFromString([@[@"set", @"Section", @"Location:", @"animated:"] componentsJoinedByString:@""]);
        if ([self.class instanceMethodForSelector:selector] == [UITableViewCell instanceMethodForSelector:selector]) {
            IMP implementation = imp_implementationWithBlock(^(UITableViewCell *_self, SLUITableViewCellSectionLocation location, BOOL animated) {
                [_self setForbiddenSectionLocation:location animated:animated];
            });
            class_addMethod(self.class, selector, implementation, method_getTypeEncoding(class_getInstanceMethod(self.class, @selector(setForbiddenSectionLocation:))));
        }
    }
    
    return self;
}
```

The default implementation for `setForbiddenSectionLocation:animated:` call the original implementation of `setSectionLocation:animated:` if it is available at runtime:

```
- (void)setForbiddenSectionLocation:(SLUITableViewCellSectionLocation)location animated:(BOOL)animated
{
    SEL selector = NSSelectorFromString([@[@"set", @"Section", @"Location:", @"animated:"] componentsJoinedByString:@""]);
    if ([UITableViewCell instancesRespondToSelector:selector]) {
        struct objc_super super = {
            .receiver = self,
            .super_class = [UITableViewCell class]
        };
        
        ((void(*)(struct objc_super *, SEL, SLUITableViewCellSectionLocation, BOOL))objc_msgSendSuper)(&super, selector, location, animated);
    }
}
```

So at runtime, the methods are called in the following way:

1. The UITableView calls `-[YourUITableViewCellSubclass setSectionLocation:animated:]` on your UITableViewCell subclass.
2. Your subclasses implementation of `-[YourUITableViewCellSubclass setSectionLocation:animated:]` forwards this message to `-[YourUITableViewCellSubclass setForbiddenSectionLocation:animated:];`, which you can implement and call super on.
3. The super implementation `-[UITableViewCell setForbiddenSectionLocation:animated:];` calls the original implementation `-[UITableViewCell setSectionLocation:animated:]` if it is available at runtime.

You can find the [Source Code](https://github.com/OliverLetterer/SLUITableViewCellSectionLocation) over at GitHub and [follow me on Twitter](https://twitter.com/oletterer) if you like. As @steipete said in his talk: Psst. Don't tell Apple.