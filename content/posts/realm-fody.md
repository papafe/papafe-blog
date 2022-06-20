---
title: "What does Realm.Fody do?"
date: 2022-06-20
draft: false
tags: [csharp, realm, fody, weaving, source generators]
---

If you have ever added the Realm .NET Nuget to one of your projects, you have probably noticed that it pulls another smaller package with it, `Realm.Fody`. This article will explain what is this package doing and why it's fundamental to Realm. 

For the developers in the audience that never heard of it, Realm is an (amazing) offline-first (mostly) mobile object database with some really nice features\*. This article is not an introduction to Realm though, nevertheless I expect it should be easy to follow along even if you've never used it before. If you're curious and want to learn more about it, the [official documentation](https://www.mongodb.com/docs/realm/introduction/) is a good place to start.  Besides, Realm open source, so you can check the source code for the .NET SDK on [Github](https://github.com/realm/realm-dotnet). 

>  **_*NOTE:_**  In my day job I am actually working on the .NET SDK of Realm, so my opinion of it may be somehow skewed :grin:

In the next section I am going to give a very small introduction to what IL weaving is and how Fody relates to it, so feel free to skip it if already know about it.

## IL weaving and Fody

The compilation of .NET source code produces `Common Intermediate Language` (*CIL* or *IL* for short) code, instead of machine-specific code. IL is a platform and CPU independent instruction set, and this allows the compiled code to be compatible with all the environments that supports the Common Language Infrastructure (CLI), such as the .NET or Mono runtimes. Then, when the IL code needs to be executed it gets compiled down to native code by the JIT compiler of the runtime. 

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
}
```

As you can see IL code is not exactly human readable, and it's also quite verbose. If you want to have an idea of how your code looks like in IL you can use a decompiler tool such as [JustDecompile](https://www.telerik.com/products/decompiler.aspx) or [ILSpy](https://github.com/icsharpcode/ILSpy).

**IL weaving** refers to the process of altering the IL code. This is an extremely powerful technique, because it allows to do things that would not be possible to do just by working with source code. By altering the IL code it's possible to inject new methods, modify existing ones, add attributes to properties and much, much more. This also allows to avoid writing repetitive code as we will see later. 


Working with IL weaving as part of the building process is quite complex though, and this is where [Fody](https://github.com/Fody/Fody) enters into the picture. Fody is an extensible tool that allows to simplify the weaving process by taking care of all the heavy lifting. For this reason it's probably the de-facto standard framework used to create weaving libraries. 

All the weaving libraries (also called weavers or addin) using Fody need to have the .Fody suffix, so it's quite easy to recognize them. As an example, if you ever had to implement `INotifyPropertyChanged` manually in a class with dozens of properties you would probably welcome weaving (and Fody) with open arms. By creating all the plumbing code directly in IL, libraries like [PropertyChanged.Fody](https://github.com/Fody/PropertyChanged) allows to inject the necessary notification code without the need to modify anything in the original source code. 

## How Realm uses weaving

Now, let's take a look at how Realm uses weaving, and so what is that Realm.Fody package doing. IL weaving is actually used in several ways in Realm, but the underlying reasoning for all of them is to greatly simplify the experience for developers using the library. 

### Properties

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

As you can see IL weaving allows to hide the disparity of treatment between managed and unmanaged objects, without the need for the developer to write any additional line of code. As an additional note, be aware that the actual weaved code is more complex than what I have shown, but it gives a good idea of how setters and getters are getting altered. 


### IRealmObjectHelper

Another important part of the weaving puzzle in Realm is `IRealmObjectHelper`, an interface that represents an helper class that is used internally. Directly from [`IRealmObjectHelper.cs`](https://github.com/realm/realm-dotnet/blob/main/Realm/Realm/Weaving/IRealmObjectHelper.cs):

```csharp
public interface IRealmObjectHelper
{
    IRealmObjectBase CreateInstance();

    void CopyToRealm(IRealmObjectBase instance, bool update, bool skipDefaults);

    bool TryGetPrimaryKeyValue(IRealmObjectBase instance, out object value);
}
```

The methods in the interface provide some helpful convenience methods used to deal with specific strongly typed classes. 
For instance, `CopyToRealm` is a method used to add an object to a realm. When this happens, all its properties need to be written too, and this is what does. If we look at a simplified implementation for the `Person` class before:

```csharp
void CopyToRealm(IRealmObjectBase instance, bool update, bool skipDefaults)
{
    ...
    SetValue("Name", name);
    ...
}

```

This method is being called after the object is managed, and practically ensures that all the property values that the object had are persisted. 

The interesting thing here is that a full implementation of `IRealmObjectHelper` is actually weaved. Differently from the properties setters and getters that are just altered, in this case everything is built from the ground up. 

In order to find this implementation, the Realm objects are actually decorated with the [`WovenAttribute`](https://github.com/realm/realm-dotnet/blob/main/Realm/Realm/Attributes/WovenAttribute.cs) during weaving:

```csharp
public class WovenAttribute : Attribute
{
    internal Type HelperType { get; private set; }

    /// <summary>
    /// Initializes a new instance of the <see cref="WovenAttribute"/> class.
    /// </summary>
    /// <param name="helperType">The type of the generated RealmObjectHelper for that class.</param>
    public WovenAttribute(Type helperType)
    {
        HelperType = helperType;
    }
}
```

When needed, the helper is then instantiated directly from `HelperType`:

```csharp
var wovenAttribute = schema.Type.GetCustomAttribute<WovenAttribute>();
var helper = (IRealmObjectHelper)Activator.CreateInstance(wovenAttribute.HelperType);
```

And there it is, a full usable weaved implementation of `IRealmObjectHelper`.

### Module initializer

Module initializers are a kinda obscure feature of .NET that allows writing initialization code for an assembly. They allow libraries to do one-time initialization when loaded, without the need for the user to explicitly call anything. It's a quite niche feature, but unfortunately it was not exposed in C#, and that led to users finding their own way of using it, like with [ModuleInit.Fody](https://github.com/fodyarchived/ModuleInit). Given the need for it, Microsoft then decided to add it in supported platform [starting from C# 9.0](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/module-initializers).

In a similar way to what is done in ModuleInit.Fody, Realm is also weaving its own module initializer. This is used to build the default schema that is used to open a realm when no schema is passed in the configuration. The use of the module initializer allows this process to happen automatically, without the need for Realm users to call any initialization code themselves. 

Without going too much into details, the weaved module intializer is calling the static method `RealmSchema.AddDefaultTypes(IEnumerable<Type> types)` with all the types that have been already weaved. This adds them to the default schema as described before. 

## Drawbacks

I hope that by reading the article this far it became clear that IL weaving is an extremely powerful technique. By making possible to alter the compiled code, it allows to greatly simplify the user experience of the users of the Realm .NET library. 

Nevertheless, weaving has some major drawbacks. As I've shown [before](#il-weaving-and-fody) IL code is quite difficult to read and comprehend for developers used to a much higher level of abstraction when programming. It requires a very specific knowledge to work with, making it difficult to maintain, especially in the context of a library whose primary target is not weaving-related. Furthermore, the generated code cannot be verified by the compiler, and it's not even possible to use the debugger with it. Finally, different weavers can interact involuntarily with each other, leading to unpredictable results.

If you want to take a look yourself at how complex the IL weaving code looks like in Realm, the main weaving code can be find in
[RealmWeaver.cs](https://github.com/realm/realm-dotnet/blob/main/Realm/Realm.Weaver/RealmWeaver.cs).


## Source Generators

One of the major (and modern) alternatives to IL weaving is [**Source Generators**](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview), that were introduced in .NET 5. Source generators allows developers to inspect user code during compilation and generate code on the fly. The generated code is then compiled with the rest of the user code. 

In a certain sense, source generators allow to "step into" the compilation process. After an initial compilation run, the source generators are passed a *compilation object*, that allows to inspect the user code using both syntax and semantic models. The generator can then generate additional code that is added to the compilation object. Finally, the compilation resumes as usual. 

This is a particularly powerful technique and presents some advantages over IL weaving: 
- Source generated code is "just" code, and so it can be easily understood by a human.
- The generated code is part of the compilation, so it can be debugged and unit tested.
- Source generators work with strings, that are much easier to deal with than IL operations and codes. 
- It's much easier to generate code that is using the latest available language features.

Overall these characteristics, in my opinion, make source generators much more maintainable than IL weaving, and lead to a greatly improved developer experience. 

Unfortunately, source generators can only add code to the compilation, but not modify it. This means that they cannot provide the same functionality that weaving does. For instance, it's not possible to modify setters and getters of properties, at least until C# won't allow [partial properties](https://github.com/dotnet/csharplang/discussions/3412). This limitation means that it is not possible to simply convert weaving libraries to source generators, and a more nuanced solution is needed.


## Final words

In this article I have tried to give a small introduction to how and why the Realm .NET SDK uses IL weaving (and so Fody). As we have seen, the main reasoning is to give developers a smoother experience when using the library, by allowing to hide quite some complexity.

Even though extremely powerful, weaving is difficult to use and to maintain for a series of reason. Source generators can be an alternative, and provide a much more pleasant library developer experience, but they are still not as versatile unfortunately. 
 