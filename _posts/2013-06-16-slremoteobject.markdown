---
layout: post
title: "SLRemoteObject: It's just an objc RPC framework for your local network"
author: oletterer
authorname: Oliver Letterer
---

{{ page.title }}
================

With this and the following blog posts, I will talk about some awesome features and the powers of the dark objc runtime and will demonstrate, how we are using it to either create a really cool software package or create a great user experience. I will start by taking a look at just another one of our open source projects: __SLRemoteObject__. First of all, SLRemoteObject is inspired by NSXPCConnection and Ole Begemanns really great research of [Remote View Controllers in iOS 6](http://oleb.net/blog/2012/10/remote-view-controllers-in-ios-6/). SLRemoteObject has been used by us internally for almost a year now and has nothing to do with any new framework any fruit company has recently announced at its yearly developer conference. In a nut shell, SLRemoteObject is an objc remote procedure call (RPC) framework, that lets you exchange messages between iOS devices in your local network in a really cool way and can be set up with simply two lines of code. To explain what that means, I think it's best to first take a look at what problem SLRemoteObject is trying to solve and how it achieves this:

<!--- end preview -->

## Introducing SLRemoteObject

Imaging the following two scenarios:

1. You are developing a turn based game with a multiplayer mode which should allow two or more players in the same network to play together.
2. You are developing some kind of system (which we are currently doing) that in addition to a global cloud needs some communication between devices in the local network and it's cheaper for you to handle this communication directly between these devices instead of communicating over let's say web sockets with the cloud.

In both of these scenarios, you need some way to send a message from one device to another device to inform it that something has happened. I will be calling the message sending device the `client device` and the message receiving device the `server device` from now on. My first approach has been to run an HTTP server on the server device and broadcast it over a bonjour services. But since both devices are running on iOS with the objc runtime, I decided to take a slightly different approach here. So let's take a look at how SLRemoteObject sends a message from the client device to the server device:

For SLRemoteObject, you first start by defining a shared objc protocol over which you want to communicate:

```
@protocol SampleProtocol <NSObject>

@required
- (void)performSomeAction;

@optional
- (void)performSomeActionWithCompletionHandler:(void(^)(NSError *))completionHandler;

@end
```

All required methods will be executed on the server device and the client device will call any optional method to trigger a remote procedure call on the server device. Note that as a convention, for each required method `doThisAndThatWithObject:`, an optional method `doThisAndThatWithObject:withCompletionHandler:` is expected. In case a required method returns any object, this object will be passed into the completionHandler as the _first_ parameter. To bootstrap the server device, you simply create an instance of `SLRemoteObjectProxy` as follows (first line of code):

```
SLRemoteObjectProxy *proxy = [[SLRemoteObjectProxy alloc]
    initWithServiceName:@"someServiceName"
    target:someTargetObjectInstance
    protocol:@protocol(SampleProtocol)
    options:nil
    completionHandler:^(NSError *error) {
    	// will be called as soon as the server device is ready so receive messages
    }];
```

* The service name must be a unique key in your local network and is used by SLRemoteObjectProxy to broadcast this service via `NSNetService` to allow easy discoverability for your client devices.
* The target will be invoked with all received objc messages.
* The protocol is the protocol over which you want to communicate.
* The options let's you specify some options like SSL or symmetric encryption. In this introduction post, I will leave encryption out but SLRemoteObject is using `Security/SecureTransport.h` and `SSLContextRef` to achieve asymmetric encryption. Feel free to take a look at the source code in case you are interested on how this works.
* The completion handler will be executed as soon as the SLRemoteObjectProxy instance is ready to receive any incoming messages.

Setting up the client device is as easy as setting up the server device (step two):

```
id<SampleProtocol> remoteObject = [SLRemoteObject
    remoteObjectWithServiceName:@"someServiceName"
    protocol:@protocol(SampleProtocol)
    options:nil];
```

* The service name must be the same as the service name you specified on your server device.
* The protocol must be the same protocol you passed into your SLRemoteObjectProxy instance. In case these protocols don't match by 100 percent, SLRemoteObject will always fail with an incompatibility error to not cause any crashes at runtime.
* You can specify the same encryption options here as well.

As soon as the SLRemoteObject instance has been created, you can start sending messages to your server device by simply calling any of the optional protocol methods on your client device:

```
[remoteObject performSomeActionWithCompletionHandler:^(NSError *error) {
    // handle the error
}];
```

which will result in the SLRemoteObjectProxy instance to execute `[target performSomeAction]` on the server device. The completion handler is called as soon as the execution of `[target performSomeAction]` has completed. So let's take a look at how the objc runtime glues all this magic together.

## Under the hood

If you take a look at the `NSObject.h` header, you will encounter the following method definitions:

```
@interface NSObject <NSObject>

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
- (void)forwardInvocation:(NSInvocation *)anInvocation;

@end
```

In case you send any unknown message to an object at runtime, the objc runtime checks if you implement these two methods on your object _before_ calling `-[NSObject doesNotRecognizeSelector:]` resulting in an exception being thrown. Via `methodSignatureForSelector:`, you return an `NSMethodSignature` instance containing the objc method signatur for this selector. This signature is needed to create an `NSInvocation` which is then being forwarded to you via `forwardInvocation:`. `SLRemoteObject` implements these two methods. Your defined protocol is used to construct the method signature

```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    struct objc_method_description methodDescription = protocol_getMethodDescription(_protocol, aSelector, NO, YES);
    
    if (!methodDescription.types) {
        [self doesNotRecognizeSelector:aSelector];
    }
    
    return [NSMethodSignature signatureWithObjCTypes:methodDescription.types];
}
```

and `-[SLRemoteObject forwardInvocation:]` is doing a ton of runtime checks to ensure your protocol is defined in the correct way. SLRemoteObject then serializes the invocation into a data blob, opens a new connection to the server device and sends this data blob over a custom defined TCP protocol. (Did I mention that SLRemoteObject is also a great sample project if you want to learn more about low level TCP communication?) Once the server devices SLRemoteObjectProxy instance receives the data blob, it constructs an `NSInvocation` and invokes it on your specified target (of course after doing a ton of runtime checks to ensure your server device is compatible for this message). After that, the SLRemoteObjectProxy grabs the returned value, serialized it and sends it back to the client device. On the client device, the SLRemoteObject instance receives the response, executes your completion handler with the returned object and closes the connection.

## Limitations

I mentionad above that the server and client devices are de/serializing `NSInvocation` instances. To make this process as easy as possible, SLRemoteObject only supports `id` typed method arguments and return types conforming to the `NSCoding` protocol.

I hope that this has been another &#x1f437; for you and in case you don't understand the pig face, you should check out the `Hidden Gems in Cocoa and Cocoa Touch` session which is simply awesome. You can find the source code over at [GitHub](https://github.com/OliverLetterer/SLRemoteObject) and find me on twitter as [@oletterer](https://twitter.com/oletterer).
