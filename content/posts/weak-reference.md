---
title: "On weak references and how to test them"
date: 2023-08-18
draft: false
tags: [csharp, gc, shorts, weakreference]
description: A short post about weak references and how to test them
---
## Introduction

When working in C#, most of the time we access objects through so-called strong references. With strong references, if the application code can still reach a certain object, then it cannot be garbage collected and, if that object is then referencing other objects, then those also cannot be collected themselves.

Weak references, instead, allow the garbage collector to freely collect objects at will to free memory space, while still allowing to access them before they get collected. Weak references are particularly useful when we need to reference objects that can potentially use a lot of memory, but can be recreated easily. Weak references can also be useful to avoid creating strong circular references that can prevent objects from being collected, [for instance in iOS](https://learn.microsoft.com/en-us/xamarin/ios/deploy-test/performance#avoid-strong-circular-references). 

In this small blog post I'll go over how to create and use a weak reference, and also how to test that it gets correctly garbage collected. If you need more information about weak references themselves, then you can take a look at the [documentation](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/weak-references). 

## How to create and use a weak reference

In C# you can use the `WeakReference` class to create a weak reference to an object. 

For example, let's imagine that we have a `Container` class with a `MainView` property. `MainView` could potentially occupy quite a lot of memory, but it's relatively easy to re-create. For this reason we prefer to keep it as a weak reference:

```csharp
public class Container
{
    private WeakReference<View>? mainViewWeakReference;

	public View MainView
	{
		get
		{
			if (mainViewWeakReference is null || !mainViewWeakReference.TryGetTarget(out var targetView))
			{
				// Possibly View could occupy a lot of memory
				targetView = new View();

				if (mainViewWeakReference is null)
				{
					mainViewWeakReference = new WeakReference<View>(targetView);
				}
				else
				{
					mainViewWeakReference.SetTarget(targetView);
				}
			}

			return targetView;
		}
	}
}
```

The `MainView` property here is used to return the contained main view, as an example. 
There are a couple of things to notice:
- We use `TryGetTarget` to check if the contained `View` has been collected already. If it has not been collected, then we just return it.
- Otherwise we create the `View` object, and then we either create the `WeakReference<View>` object, or we change the target object contained in `viewWeakReference` with `SetTarget`.

After the getter is called, the GC is free to collect the contained view when there are not other strong references to it. 

Another small thing to notice is that we actually have a strong reference to `WeakReference` in the code, but the `View` object inside it is stored as a weak reference. 

## How to test a weak reference

Writing a unit test to verify if the target of a weak reference is garbage collected correctly is a little bit tricky, because in order for that to happen there shouldn't be any strong reference to it. For this reason, the easiest way to create a unit test is to make a weak reference to the object that we expect to be collected in a delegate or another method, for example in an `Action` or `Func`, like in the following example:

```csharp
[Test]
public void VerifyThatViewIsGarbageCollected()
{
    var container = new Container();

    var wr = new Func<WeakReference>(() =>
    {
        return new WeakReference(container.MainView);
    })();

    GC.Collect();

    Assert.That(wr.IsAlive, Is.False);
}
```

This test is verifying that `Container.MainView` is garbage collected. To do so, we create a weak reference to it in the `Func`, and then return it. We then force garbage collection by calling `GC.Collect`, and then we verify that the target has been garbage collected with `IsAlive`. In this case we are creating the weak reference inside the `Func` delegate, so that after the delegate is called, there will be no actual strong reference to `MainView`, so that we can verify it gets actually collected.

As a final note, please remember that the way that garbage collection works is slightly different ways when working in [Debug or Release mode](https://stackoverflow.com/questions/7165353/does-garbage-collection-run-during-debug), so that's something to take into considerations when testing. 