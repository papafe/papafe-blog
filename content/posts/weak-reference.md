---
title: "On weak references and how to test them"
date: 2023-08-15
draft: true
tags: [csharp, gc, shorts]
description: TODO
---
## Introduction

Most of the time, when working with C# we access objects through so-called strong references. When the application code can still reach a certain object, then it cannot be garbage collected. If that object is then referencing other objects, then those also cannot be collected themselves.

Weak references, instead, allow the garbage collector to freely collect objects at will to free memory space, while still allowing to access them before they get collected. Weak references are particularly useful when we need to reference objects that can potentially use a lot of memory, but can be recreated easily. Weak references can also be useful to avoid creating strong circular references that can prevent objects from being collected, [for instance in iOS](https://learn.microsoft.com/en-us/xamarin/ios/deploy-test/performance#avoid-strong-circular-references). 

In this small blog post I'll go over how to create and use a weak reference, and also how to test that it gets correctly garbage collected. If need more information about weak references themselves, then you can take a look at the [documentation](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/weak-references). 

## How to create and use a weak reference

In C# you can use the `WeakReference` class to create a weak reference to an object. 

For example, let's imagine that we have a `Container` class that can create a `View`, but should not keep a strong reference it, as it could potentially occupy quite a lot of memory, and it should not be create beforehand. For this reason it keeps a weak reference to the `View`:

```csharp
public class Container
{
    private WeakReference<View>? viewWeakReference;

    public View GetView()
    {
        if (viewWeakReference is null || !viewWeakReference.TryGetTarget(out var targetView))
        {
            // Possibly View could occupy a lot of memory
            targetView = new View();

            if (viewWeakReference is null)
            {
                viewWeakReference = new WeakReference<View>(targetView);
            }
            else
            {
                viewWeakReference.SetTarget(targetView);
            }
        }

        return targetView;
    }
}
```

The `GetView` method here is used to return the contained view, as an example. 
There are a couple of things to notice:
- We use `TryGetTarget` to check if the contained `View` has been collected already. If it has not been collected, then we just return it.
- Otherwise we create the `View` object, and then we either create the `WeakReference<View>` object, or we change the target object contained in `viewWeakReference` with `SetTarget`.

After this method is called, the GC is free to collect the contained view when there are not other strong references to it. Another small thing to notice is that we actually have a strong reference to `WeakReference` in the code, but the `View` object inside it is stored as a weak reference. 

## How to test a weak reference

What are weak references
Strong references vs weak references

A little bit about garbage collection
How to test weak references collection (create an action, for instance)