---
title: "What does Realm.Fody do?"
date: 2022-06-01T22:46:34+02:00
draft: false
tags: [Space, mat]
categories: [my-category]
---
If you have ever added the Realm .NET nuget in one of your projects, you have probably noticed that it pulls another smaller package with it, `Realm.Fody`. This article will explain what is this package doing and why it's fundamental to Realm. 

In the next section I am going to give a very small introduction to what IL weaving is and how Fody relates to it, so feel free to skip it if already know about it.

## IL weaving and Fody

The compilation of .NET source code produces Common Intermediate Language (CIL or IL for short) code, instead of machine-specific code. IL is a platform and CPU independent instruction set, and this allows the compiled code to be compatible with all the environments that supports the Common Language Infrastructure (CLI), such as the .NET or Mono runtimes. Then, when the IL code needs to be executed it gets compiled down to native code by the JIT compiler of the runtime. 
> **_NOTE:_**  This is a simplification, and some of the steps can be different, but it gives a good picture of the process.

I realise that it can be a little though to follow trough the process and all those acronyms, but the main takeaway is that the source code gets transformed into a series of instructions after compilation. To give an idea of how IL looks like, if we take this simple `HelloWorld` method here:

```csharp
public static void HelloWorld(string name)
{
    Console.WriteLine($"Hello {name}! How are you?");
}
```

The compiler produces this:

```
method public hidebysig static void HelloWorld (string name) cil managed 
{
    .maxstack 8

    IL_0000: nop
    IL_0001: ldstr "Hello "
    IL_0006: ldarg.0
    IL_0007: ldstr "! How are you?"
    IL_000c: call string [mscorlib]System.String::Concat(string, string, string)
    IL_0011: call void [mscorlib]System.Console::WriteLine(string)
    IL_0016: nop
    IL_0017: ret
} // end of method MainClass::HelloWorld
```

As you can see IL code is not exactly human readable, and it's also quite verbose. If you want to have an idea of how your code looks like in IL you can use a decompiler tool such as [JustDecompile](https://www.telerik.com/products/decompiler.aspx) or [ILSpy](https://github.com/icsharpcode/ILSpy).

**IL weaving** refers to the process of altering the IL code. This is an extremely powerful technique, because it allows to do things that would not be possible to do just by working with source code. By altering the IL code it's possible to inject new methods, modify existing ones, add attributes to properties and much, much more. This also allows to avoid writing repetitive code as we will see later. 


Working with IL weaving as part of the building process is quite complex though, and this is where [Fody](https://github.com/Fody/Fody) enters into the picture. Fody is an extensible tool that allows to simplify the weaving process by taking care of all the heavy lifting. For this reason it's probably the de-facto standard framework used to create weaving libraries. 

As an example, if you ever had to implement `INotifyPropertyChanged` manually in a class with dozens of properties you would probably welcome weaving (and Fody) with open arms. By creating all the plumbing code directly in IL, libraries like [PropertyChanged.Fody](https://github.com/Fody/PropertyChanged) allows to inject the necessary notification code without the need to modify anything in the original source code. 


## Realm.Fody

Realm.Fody uses weaving for a series of reasons. The main one is that we want to weave properties. 
If you've used Realm before then you know that realm classes look like POCOs, for instance:

```sharp
class Person
{

}
```

The good thing about Realm is that we hide all of the complexity, but managed and unmanaged objects need to be treated differently.
For unmanaged objects setting/getting the value means setting/getting the backing fiedl
For managed objects setting/getting a property means to read/write the value directly from/to the database. 

In order to have a unified experience, and to be able to treat the object in the same way independently from the state, the weaved class will look like something like:

```sharp
class Person
{

}
```

As you can see....

Another important thing is the module init. This is used to build the default schema. It works like this:



## Drawbacks
Very powerful, but it has several drawbacks too:
- IL weaving code is extremely difficult to read
- It requires a very specialized knowledge, it's error prone and trial and error
- It's not debuggable
- It happens after compilation, so the compiler cannot verify correctness, or even optimize the code
- Very difficult to test


## Alternatives

One of the alternatives that we're going to use is called source generators. 
What they do is that they add code to the compilation, they don't modify the pre-existing code like weaving does. 
For this reason (they are only additive), they are also more limited in what can be achieved with them.
But:
- SG generates code, that can be read and understood by a "human". It doesn't require specialized knowledge
- The generated code is part of the compilation, so it can be debugged
- SG code is just text, so it's quite easy to test source generators, because we can just compure the sg code and what we expected
- The code is analyzed by the compiler, that can eventually optimize it and verify correctness

Unfortunately given its additive nature not everything can be done with them

## Final words



Adam Furmanek videos on IL

Fody
ILSpy
JustDecompile
