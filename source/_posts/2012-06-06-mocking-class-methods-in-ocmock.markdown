---
layout: post
title: "Class Methods in OCMock"
date: 2012-06-06 16:09
comments: true
categories: [UnitTesting, MockObjects, OCMock, Objective-C]
---

I'm opening my blog with a post about patches I've made to [OCMock](http://ocmock.org) to handle mocking and stubbing class methods. You can find my fork [HERE](https://github.com/kykim/ocmock).

I built on top of the work in progress in the OCMock master. It works, but will probably need some refinement.

Let's the example of a static constructor in `NSString`:
    NSString *s = [NSString string];    // returns empty string

In order to mock this, we do this:
    id mockStringClass = [OCMockObject mockForClassObject:[NSString class]];
    NSString *mockString = @"Mocked String Result";
    [[[mockStringClass stub] andReturn:mockString] string];
    NSString *actualString = [NSString string];
    STAssertTrue(0 != [actualString length], @"Expected String Not Be Empty");

I sent the pull request about two weeks ago, still waiting to hear back. Read on if you're interested in the details...

<!-- more -->

[OCMock](http://ocmock.org) is one of the more popular Objective-C/iOS [mocking](http://www.mockobjects.com/) frameworks out there. It actually predates iOS.

One of the biggest problems I've always had with OCMock was its inability to handle class methods. There are many times when I've had to call a class method, that for various reasons I needed to mock.

``` objc Class Methods in iOS MediaPlayer framework
MPMediaQuery *songsQuery = [MPMediaQuery songsQuery];
NSArray *songs = [songsQuery items];
```

If you're running this on the simulator, you don't have a media library (aka iTunes library). So how do you test your code? Besides, testing against "live" data is probably a bad idea.

Looking at the OCMock [Github](http://github.com/erikdoe/ocmock) repository, there was some preliminary work done on supporting class methods. The class is named OCMockClassObject ([header](https://github.com/erikdoe/ocmock/blob/master/Source/OCMock/OCMockClassObject.h), [source](https://github.com/erikdoe/ocmock/blob/master/Source/OCMock/OCMockClassObject.m)). So why doesn't this work?

Looking into the unit tests for OCMock, I found the tests for OCMockClassObject. Interestingly, one of the tests,
    // - (void)testForwardsUnstubbedMethodsToRealClassObjectAfterStopIsCalled
is commented out. Why? First, let's understand how OCMock works.

When you mock a class, you declare something like this:

``` objc Mocking an instance method with OCMock
id mockObject = [OCMockObject mockForClass:[NSString class]];
[[mockObject expect] andReturn:@"mockedstring"] lowercaseString];
... // code to test
[mockObject verify];
```

`mockObject` is declared as mock for `NSString`. Also, we state that we *expect* the method `lowercaseString` to be invoked on `mockObject`. Furthermore, `lowercaseString` should return "mockedstring". We defined some test code where `mockObject` is used. Typically, it is passed into the method we are testing. Finally, we invoke `[mockObject verify]` to confirm that `lowercaseString` was called. If not, the test fails.

Without delving too deeply into the specifics of the OCMock, how are these method calls intercepted? Via [NSProxy](https://developer.apple.com/library/mac/#documentation/Cocoa/Reference/Foundation/Classes/NSProxy_Class/Reference/Reference.html).

In short, `NSProxy` is an abstract class, and requires its subclasses to implement
    - (void)forwardInvocation:(NSInvocation *)anInvocation`
An `OCMock` class (`OCMockRecorder` specifically) implements `forwardInvocation` to handle the method call as specified. In the example above, the call is forwarded to a return the value specified in the `andReturn:` call.

For other mocking options in `OCMock`
    + (id)mockForProtocol:(Protocol *)aProtocol;
    + (id)partialMockForObject:(NSObject *)anObject;
    + (id)niceMockForClass:(Class)aClass;
    + (id)niceMockForProtocol:(Protocol *)aProtocol;
    + (id)observerMock;
the forwardInvocation relies on [method swizzling](http://cocoadev.com/wiki/MethodSwizzling) to redirect methods to expectations and/or stubs.

So why doesn't this work for class methods? Well, it turns out it does. The real problems is resetting back to the original state.

One of the important features of unit testing is to make sure each test is atomic. That is, the results on one test should not impact the results of another. To that end, we use `setUp` and `tearDown` methods to initialize and clean up our environment before and after running each test. Furthermore, we may do additional configuration inside each test, and it's incumbent upon us as developers to make sure we clean up after ourselves.

Looking back at the `OCMock` source, we can see that the mocking of class methods works fine. It's just that our swizzled class method stays swizzled. Why does it work for instance methods? Because the instance is destroyed with each test (hopefully), so any evidence of method swizzling just disappears with the instance. Class method swizzling will persist for the run life of the application (or in this case, the test harness). So we need a away to undo our swizzling.

My solution was simple: create an `NSMutableDictionary` to store the [IMP pointer](https://developer.apple.com/library/ios/#documentation/Cocoa/Reference/ObjCRuntimeRef/Reference/reference.html#jumpTo_96) to the original class method, keyed on the string representation of the class method selector. When the mocked class object is deallocated, simply iterate over the dictionary, replacing all the swizzled class methods for their originals.

I'm making the assumption that the location of the method (and class definintion) pointed to by the `IMP` pointer won't change during the run life of the application/test harness. I figured this was unlikely, but I needed to make sure.

Who better to ask than Mr. Objective-C: [Steve Naroff](http://www.linkedin.com/pub/steve-naroff/4/671/194)? His response:

    In theory, no. In practice, yes.

    If the code were dynamically loaded/unloaded, the address could
    change. If the code were compiled on the fly, you could imagine
    the runtime purging infrequently used code and re-instantiating it
    later if necessary (to keep the working set of methods down on a
    smaller footprint device). Unless things have changed a great deal
    in the past 2 years, I'd be surprised if my scenarios happen often
    (or at all).

Well, I'm pretty sure that won't happen while running tests, so this should be safe.

This can be pretty esoteric stuff, that most iOS won't ever have to deal with, but to make testing frameworks, know the Objective-C runtime is pretty essential. I recommend the following for more information:

* [The Objective-C Programming Language](https://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/ObjectiveC/Introduction/introObjectiveC.html)
* [Objective-C Runtime Programming Guide](https://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html)
* [Objective-C Runtime Reference](https://developer.apple.com/library/ios/#documentation/Cocoa/Reference/ObjCRuntimeRef/Reference/reference.html)

