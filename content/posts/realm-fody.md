---
title: "What does Realm.Fody do?"
date: 2022-06-01T22:46:34+02:00
draft: false
tags: [Space, mat]
categories: [my-category]
---
If you have ever added the Realm .NET Nuget to one of your projects, you have probably noticed that it pulls another smaller package with it, `Realm.Fody`. This article will explain what is this package doing and why it's fundamental to Realm. 

For the developers in the audience that never heard of it, Realm is an (amazing) offline-first (mostly) mobile object database with some really nice features\*. This article is not an introduction to Realm though, nevertheless I expect it should be easy to follow along even if you've never used it before. If you're curious about it and want to learn more about it, the [official documentation](https://www.mongodb.com/docs/realm/introduction/) is a good place to start. Besides, Realm is also open source, so you can check the source code for the .NET SDK on [Github](https://github.com/realm/realm-dotnet). 

>  **_*NOTE:_**  In my day job I am actually working on the .NET SDK of Realm, so my opinion of it may be somehow skewed :grin:

In the next section I am going to give a very small introduction to what IL weaving is and how Fody relates to it, so feel free to skip it if already know about it.

## IL weaving and Fody

The compilation of .NET source code produces Common Intermediate Language (CIL or IL for short) code, instead of machine-specific code. IL is a platform and CPU independent instruction set, and this allows the compiled code to be compatible with all the environments that supports the Common Language Infrastructure (CLI), such as the .NET or Mono runtimes. Then, when the IL code needs to be executed it gets compiled down to native code by the JIT compiler of the runtime. 

I realize that it can be a little though to follow trough the process and all those acronyms, but the main takeaway is that the source code gets transformed into a series of instructions after compilation. To give an idea of how IL looks like, if we take this simple `HelloWorld` method here:

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

All the weaving libraries (also called weavers or addin) using Fody need to have the .Fody suffix, so it's quite easy to recognize them. As an example, if you ever had to implement `INotifyPropertyChanged` manually in a class with dozens of properties you would probably welcome weaving (and Fody) with open arms. By creating all the plumbing code directly in IL, libraries like [PropertyChanged.Fody](https://github.com/Fody/PropertyChanged) allows to inject the necessary notification code without the need to modify anything in the original source code. 

## Realm.Fody

Now, let's take a look at why Realm uses weaving, and so what is that Realm.Fody package doing. IL weaving is actually used in several ways in Realm, but the underlying reasoning for all of them is to greatly simplify the experience for developers using the library. 

One of the main points for weaving is simplifying the class definition. In Realm, classes are simple POCOs that derive from `RealmObject` or `EmbeddedObject`. For instance:

```csharp
class Person: RealmObject
{
    public string Name { get; set; }
}
```
When using an object from the `Person` class, developers are able to use it in the same way wether the object is *managed* (it has been added to a realm) or not (*unmanaged*). But there is a huge difference in the way properties are accessed depending on the state of the object.

If the object is unmanaged, then using the properties means just using the associated backing fields, and not much more. If the object is managed, instead, the situation is more complex. Realm actually has a zero-copy architecture, and every time the value of a property is needed, the value is retrieved directly from the database itself. Similarly, assigning the value of a property means writing it directly to the database (in this case this needs to happen in a write transaction, but that's a story for another time).

This kind of complexity is what is being hidden with IL weaving. The `Person` class you've seen before becomes something like the following after weaving:

```csharp
class Person: RealmObject
{
    private string name;

    public string Name
    {
        get
        {
            if(isManaged)
                return GetValue("Name");
            else
                return name;
        }
        set
        {
            if(isManaged)
                SetValue("Name", value);
            else
                name = value;
        }
    }
}
```

Here `GetValue` and `SetValue` are two methods defined on `RealmObjectBase` (the base class of `RealmObject`) that allow to retrieve(write) a value directly from(to) the database. 

As you can see IL weaving allows to hide the disparity of treatment between managed and unmanaged objects, without the need for the developer to write any additional line of code. 

As an additional note, please be aware that the weaved code I posted is actually more complex, but it gives a good idea of how setters and getters are getting altered. 





## CopyToRealm

ObjectHelper or whatever it is called

## ModuleInit


Another important thing is the module init. This is used to build the default schema. It works like this:


## Drawbacks
Horrible to look at (add some gists). Here I've posted code, but the weaved code is actually not visible

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
