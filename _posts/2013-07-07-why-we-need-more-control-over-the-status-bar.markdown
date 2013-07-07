---
layout: post
title: "Why we need more control over the status bar in iOS 7"
author: oletterer
authorname: Oliver Letterer
---

{{ page.title }}
================

Starting with iOS 7, Apple is going to introduce one major change to the status bar in every application: All the obstacles and borders of the status bar will disappear and your content is going to be presented full screen by default. [@steipete](https://twitter.com/steipete) as already enlightened us on a way how more status bar control can improve the overall UX in his [UIKonf talk](http://www.uikonf.com/speakers/peter_steinberger.html) and with this blog post, I'm going to discuss why we need more control over the status bar starting with iOS 7 more then ever to achieve a great UX.

<!--- end preview -->

Image of the status bar under iOS 6:
![image iOS 6](/images/screenshot_ios_6.PNG)

Image of the status bar under iOS 7:
![image iOS 7](/images/screenshot_ios_7.PNG)

## The new status bar and existing UI concepts

Before iOS 7, the status bar has just been some third party view hovering above the application. But starting with iOS 7 and the fact that all borders and obstacles have been removed, the user gets the impression that the status bar is an actual part of the content being presented and the application __wants__ to display the status bar here in that place. The navigation and status bar are forming some kind of union and are melting together.

A common navigation concept in many popular iOS applications is that of a slide to reveal or side panel view controller like [PKRevealController](https://github.com/pkluz/PKRevealController) (we have of course written our own side panel view controller at some point in time but we won't open source it because there are already enough good implementations out there):

![Side panel under iOS 6](/images/slide_ios_6.gif)

which works pretty great under iOS 6 and below to give the user an easy way to access different navigation stacks when needed. In comparison to a `UITabBarController`, these navigation stacks are hidden most of the time to leave room for the main content. Now let's take a look at what this navigation concept will look like under iOS 7 by default:

![Side panel under iOS 7](/images/slide_ios_7.gif)

The minor problem here is (I will come to the major one in a second) obviously that the status bar is overlapping with the side panel view controller. One could fix this applying a contentInset to the table view controller and/or display a blurred view between the side panel and the status bar. But this would be just a workaround for the main problem: The overall illusion that the navigation and status bar are belonging together is totally broken at this point and the status bar is just becoming a third party view hovering over your application again. Furthermore, revealing the side panel view controller is kind of a modal action where the user __doesn't__ want to interact with his main contont nor does he need the status bar; just change the main content real quick. That is why the status bar should move _with_ the main content view controller:

![Side panel and status bar under iOS 7](/images/slide_ios_7_with.gif)

Moving the status bar _with_ the main content view controller is giving the user more space to focus on the side panel and the status bar is displayed where it makes most sense: while interacting with the main content.

## Private API

The problem is of course that this is not possible without private APIs. One could simply get and translate the status bar

``` objc
UIView *statusBarView = [[UIApplication sharedApplication] valueForKey:[@[@"status", @"Bar"] componentsJoinedByString:@""]];
statusBar.transform = CGAffineTransformMakeTranslation(translation, 0.0f);
```

but there is no guarantee that Apple won't reject any application doing this. I filed a radar on this [rdar://14370134](http://openradar.appspot.com/radar?id=3113416) and if you like to help and agree with me, consider duplicating it. As always, you can find me on twitter as [@oletterer](https://twitter.com/oletterer) for further discussion.
