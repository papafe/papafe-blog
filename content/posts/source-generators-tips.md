---
title: "Tips and tricks for source generators"
date: 2022-06-20
draft: true
tags: [csharp, roslyn, source generators]
---
# Introduction

[`Source Generators`](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview) are an amazing feature of the .NET Compiler Platform (Roslyn) added in .NET 5 that allows developer to "step into" the compilation, inspect the user code and eventually generate additional code on the fly.
Code generation makes possible, among other things, to automate writing boilerplate/repetitive code, to replace reflection in code discovery and much more. 

Source generators are also considerably more "developer friendly" than other code generation technologies, such as IL Weaving (which we've talked in ....). The main disadvantage over IL Weaving though, is that source generators can only add code to the compilation, and not modify the existing one. However, this limitation is definitely counterbalanced by its relative ease of use in my opinions

This article is a collection of information that I have gathered while working on adding a source generator to the .NET SDK of Realm (LINKKKK). This is not an introduction to source generators, as I think there is already a lot of good introductory material out there. 

Being a "relatively" new feature and dealing with the Roslyn SDK
(that is quite complex in itself), I found the information available on how to write a source generator to be quite sparse. For this reason this article is mostly a collection of various tips, tricks and considerations that I have gathered over time, and that I hope will be useful to someone else too. 

Please note that this was my first time working on source generators, and I didn't have any experience with analyzers or Roslyn beforehand, so it's very well possible that some of my suggestions do not follow the recommended approach, or there are some caveats that I did not anticipate. Definitely leave a comment if you think there is something that can be improved.


# Writing



SharpLab online to check syntax trees (or maybe use the vs studio thing already)

Generally I found convenient to have access to both the syntax and the semantic model for something. It's true that there is not necessarily a 1-to-1 relationship, but it's pretty convenient. 

You can also always go from syntax to semantic if you have the context, but not viceversa (and it makes sense, since the "class" can be defined in multiple places, for instance if partial). 

## Properties

```csharp
public static bool IsAutomaticProperty(this PropertyDeclarationSyntax propertySyntax)
{
    // This means the property has explicit getter and/or setter
    if (propertySyntax.AccessorList != null)
    {
        // Body is "classic" curly brace body
        // ExpressionBody is =>
        return !propertySyntax.AccessorList.Accessors.Any(a => a.Body != null | a.ExpressionBody != null);
    }

    // This means the body is => (propertySyntax.ExpressionBody != null)
    return false;
}

public static bool HasSetter(this PropertyDeclarationSyntax propertySyntax)
{
    return propertySyntax.AccessorList.Accessors.Any(SyntaxKind.SetAccessorDeclaration);
}
```

## Nullability

There is a difference when the nullability annotations are enabled or not
- None. No nullability at all. This is for reference types when nullability annotations are not on
- Annotated/NotAnnotated. This is for value types (all the time) and reference types (only when nullability annotations are no)

Remember that nullability annotations can be turned on even just at a local level with directives like #nullable enable

## Attributes

```csharp

public static bool HasAttribute(this ISymbol symbol, string attributeName)
{
    return symbol.GetAttributes().Any(a => a.AttributeClass.Name == attributeName);
}

public static object GetAttributeArgument(this ISymbol symbol, string attributeName)
{
    var attribute = symbol.GetAttributes().FirstOrDefault(a => a.AttributeClass.Name == attributeName);
    return attribute?.ConstructorArguments[0].Value;
}

```

## Interfaces

Remember that allInterfaces returns all the interfaces implemented by this type and all his super types, so it can be a long list, and maybe not what you're interested into. 

```csharp
public static bool IsRealmObject(this ITypeSymbol symbol)
{
    return symbol.AllInterfaces.Any(i => i.Name == "IRealmObject");
}

```


#??Debugging

Before it was a pain in the ass, now it's easier to debug.

Suggestion: create a new project and add your source generator as a reference like: ....
Then create a profile using the following;

But now we can use https://turnerj.com/blog/csharp-source-generator-pain-points-february-2022-update. () (remember to add .NET Compiler Platform SDK, https://github.com/dotnet/roslyn-sdk/issues/850). Then you can add a roslyn profile, and select a target object. You can then compile everytime it goes to generate that project

It's extremely useful, especially for debugging diagnostics, as those are kinda difficult to use

The issue is that there is still some problems with caching sgs that are in the same project. 

You can emit projects like:..... 
Those are better than the ones you get under the analyzer folders, but still they have some issues





# Testing

Testing the functionality of your code is something you definitely need to do (and that we do already), for which you don't need to look at how the generated code looks like, but only if it behaves how you would expect. 

If you want to test also how the generate code looks like and (much more important in my opinion), if the diagnostics are generated correctly and in the right place, they you can use what's in the roslyn cookbook. 

In the debugging section I have talked about creating a playground project to see how the source generators produces output and diagnostic. The actual tests can actually refer to those, so it's much easier. Remember, this is only for how the code/diagnostics looks like. You'll need to have something different in place for what it concerns how the code works.


Some tips:
- How to avoid generating the usual compiler diagnostics (we don't care about those diagnostics, only the ones that we create)
- How to add a reference to your project



