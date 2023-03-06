---
title: "Tips and tricks for source generators"
date: 2022-06-20
draft: true
tags: [csharp, roslyn, source generators]
---
# Introduction

[`Source Generators`](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview) are an amazing feature of the .NET Compiler Platform (Roslyn) added in .NET 5 that allows developer to "step into" the compilation, inspect the user code and generate additional code on the fly.
Code generation makes possible, among other things, to automate writing boilerplate/repetitive code, to replace reflection in code discovery and much more. 

Source generators are also considerably more "developer friendly" than other code generation technologies, such as IL Weaving (which I have talked in another [blog post]( {{< relref "realm-fody.md" >}} ). The main disadvantage over IL Weaving though, is that source generators can only add code to the compilation, and not modify the existing one. However, this limitation is definitely counterbalanced by its relative ease of use in my opinion.

This article is a collection of information that I have gathered while working on adding a source generator to the [.NET SDK of Realm](https://github.com/realm/realm-dotnet). This is not an introduction to source generators, as I think there is already a lot of good introductory material out there. 

Being a "relatively" new feature and dealing with the Roslyn SDK
(that is quite complex in itself), I found the information available on how to write, debug and test a source generator to be quite sparse. For this reason this article is mostly a collection of various tips, tricks and considerations that I have gathered over time, and that I hope will be useful to someone else too. I have divided the tips in different topics, so it should be easier to find something you are interested into.

This was my first time working on source generators, and I didn't have any experience with analyzers or Roslyn beforehand, so it's very well possible that some of my suggestions do not follow the recommended approach, or there are some caveats that I did not anticipate. Definitely leave a comment if you think there is something that can be improved. 

As a final note before going further, if you are looking for how to do certain things with source generators, it's always a good idea to take a look first at the [Roslyn Source Generator Cookbook](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md), that contains guidelines for quite a number of common patterns. 


# Writing

## Move between syntax and semantic 

When inspecting the user code, there are essentially two different levels that can be probed to get information, the syntactic level and the semantic level.

At the syntactic level the user code is represented by a syntax tree with the compilation unit at the root and the nodes (classes derived from `SyntaxNode` in the RoslynAPI) of the tree being the different elements of the code, such as properties declarations, using directives or string literals, for instance. The syntax tree contains all the information about how the user code is laid out, so it's possible to rebuild the source code exactly from it. 

At the semantic level the code is represented by a series of symbols (classes implementing `ISymbol` in the RoslynAPI) and their relationships, representing types, namespaces, properties and so on. Differently from the syntax nodes, the symbols and are related to the *meaning* of the code, and not how it is structured. At this level, for instance, it's possible to find the namespace in which a certain type is defined, or find information about base classes and overridden methods. 
Even though the semantic model is more powerful than the syntactic level I found myself using both of them for different reasons, depending on how easy it was to retrieve the information I needed for a specific case. Fortunately it's quite easy to move from one to the other.

### From syntax to semantic (`SyntaxNode` -> `ISymbol`)

```csharp
let symbol = context.SemanticModel.GetDeclaredSymbol(syntaxNode) //context is GeneratorExecutionContext
```

There are actually various overloads of the `GetDeclaredSymbol` method depending on the kind of input `syntaxNode`, and you can find more info in the [docs](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.csharpextensions.getdeclaredsymbol?view=roslyn-dotnet). 


### From semantic to syntax (`ISymbol` -> `SyntaxNode`)

Symbol -> Node 

```csharp
var references = symbol.DeclaringSyntaxReferences //returns ImmutableArray<SyntaxReference> 
foreach (SyntaxReference sr in references)
{
    var syntaxNode = sr.GetSyntax()
} 
```

`ISymbol.DeclaringSyntaxReferences` can return multiple values (for example with partial class declaration) or none if declared externally. Additionally, `SyntaxReference.GetSyntax()` can trigger a parse of the syntax tree to recover the corresponding node.

## Explore the syntax tree

It is definitely useful to check how the syntax tree looks like for certain code, in order to understand how to retrieve a certain kind of info. For this operation I definitely recommend taking a look at [`SharpLab.io`](https://sharplab.io/), a .NET code playground that contains also a syntax visualizer. While is it possible to get a [syntax visualizer in Visual Studio too](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/syntax-visualizer?tabs=csharp), I personally preferred to have an external tool to do check syntax trees, in order to avoid disrupting the current development flow. 

## Properties

A collection of utility extension methods regarding properties.

### Check if a property is automatic
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

    return false;
}
```

### Check if a property has a setter/getter
```csharp
public static bool HasSetter(this PropertyDeclarationSyntax propertySyntax)
{
    return propertySyntax.AccessorList.Accessors.Any(SyntaxKind.SetAccessorDeclaration);
}

public static bool HasGetter(this PropertyDeclarationSyntax propertySyntax)
{
    return propertySyntax.AccessorList.Accessors.Any(SyntaxKind.GetAccessorDeclaration);
}
```

## Attributes

A collection of utility extension methods regarding attributes.

### Check if symbol has a certain attribute
```csharp

public static bool HasAttribute(this ISymbol symbol, string attributeName)
{
    return symbol.GetAttributes().Any(a => a.AttributeClass.Name == attributeName);
}
```
### Get the arguments of an attribute

```csharp
public static object GetAttributeArgument(this ISymbol symbol, string attributeName)
{
    var attribute = symbol.GetAttributes().First(a => a.AttributeClass.Name == attributeName);
    //This will return only the first argument, but it can be easily generalized
    return attribute?.ConstructorArguments[0].Value;
}
```
## Interfaces

A collection of utility extension methods regarding interfaces.

### Check if a type implements an interface
```csharp
public static bool Implements(this ITypeSymbol symbol, string interfaceName)
{
    return symbol.AllInterfaces.Any(i => i.Name == interfaceName);
}
```

### Check if a type directly implements an interface
```csharp
public static bool DirectlyImplements(this ITypeSymbol symbol, string interfaceName)
{
    return symbol.AllInterfaces.Any(i => i.Name == interfaceName);
}
```

## Mixed

A collection of mixed utility extension methods.

### Check if a class is partial

```csharp
public static bool IsPartial(this ClassDeclarationSyntax cds)
{
    return cds.Modifiers.Any(m => m.IsKind(SyntaxKind.PartialKeyword))
}
```
## Nullability

The nullability of a certain type can be found by checking [`ITypeSymbol.NullableAnnotation`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.itypesymbol.nullableannotation?view=roslyn-dotnet#microsoft-codeanalysis-itypesymbol-nullableannotation).
The value of this enum is dependent on wether the nullability annotations are enabled or not:
- `None`. No nullability information at all. This is only for reference types when nullability annotations are not enabled;
- `Annotated`/`NotAnnotated`. These are used for value types (with nullability annotations enabled or not) and reference types (only when nullability annotations are on). `Annotated` means that the type has a `?`.

Remember that nullability annotations can be turned on at a project level as well as at a local level with directives like `#nullable enable` (more info in the [docs](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references)).

# Debugging

Debugging source generators used to be a really painful experience, but fortunately the situation greatly improved from [Visual Studio v16.10](https://learn.microsoft.com/en-us/visualstudio/releases/2019/release-notes-v16.10#NETProductivity) that introduced first class debugger support for source generators. 

In order to use the debugger support you need to add the `<IsRoslynComponent>true</IsRoslynComponent>` to the project file of the source generator project and be sure to have the `.NET Compiler Platform SDK` component installed. 

At this point my suggestion would be to create a project that can become your *playground* for debugging (and testing) the source generator. After creating the playground project, you should add a reference to your source generator project and edit the `csproj` file like this:

```xml
<ItemGroup>
    <ProjectReference Include="pathToTheSourceGenerator" OutputItemType="Analyzer" ReferenceOutputAssembly="false" />
</ItemGroup>

```

Then create a profile using the following (NEED TO SEE FROM MY MACHINE HOW TO DO IT)
 Then you can add a roslyn profile, and select a target object. You can then compile everytime it goes to generate that project

It's extremely useful, especially for debugging diagnostics, as those are kinda difficult to use

The issue is that there is still some problems with caching sgs that are in the same project. 

You can emit projects like:..... 
Those are better than the ones you get under the analyzer folders, but still they have some issues (need to add them and remove)


# Testing

Testing the functionality of your code is something you definitely need to do (and that we do already), for which you don't need to look at how the generated code looks like, but only if it behaves how you would expect. 

If you want to test also how the generate code looks like and (much more important in my opinion), if the diagnostics are generated correctly and in the right place, they you can use what's in the roslyn cookbook. 

Diagnostics created are different from the ones that the roslyn test thing uses, so you need some conversion. It's convenient to test it by saving it in the file system

In the debugging section I have talked about creating a playground project to see how the source generators produces output and diagnostic. The actual tests can actually refer to those, so it's much easier. Remember, this is only for how the code/diagnostics looks like. You'll need to have something different in place for what it concerns how the code works.


Some tips:
- How to avoid generating the usual compiler diagnostics (we don't care about those diagnostics, only the ones that we create)
- How to add a reference to your project
- Add a reference to framework libraries



# Others

Suppress warnings on generated code (put pragmas around file) -- Auto - generated?

