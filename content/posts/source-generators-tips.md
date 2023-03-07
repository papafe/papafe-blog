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
## Nullability in the user code

The nullability of a certain type can be found by checking [`ITypeSymbol.NullableAnnotation`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.itypesymbol.nullableannotation?view=roslyn-dotnet#microsoft-codeanalysis-itypesymbol-nullableannotation).
The value of this enum is dependent on wether the nullability annotations are enabled or not:
- `None`. No nullability information at all. This is only for reference types when nullability annotations are not enabled;
- `Annotated`/`NotAnnotated`. These are used for value types (with nullability annotations enabled or not) and reference types (only when nullability annotations are on). `Annotated` means that the type has a `?`.

Remember that nullability annotations can be turned on at a project level as well as at a local level with directives like `#nullable enable` (more info in the [docs](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references)).

## Nullability in the generated code

By default, the generated files will have nullability annotations disabled, wether or not the annotations are enabled in the project in which the file are generated. For this reason it's better to specify manually the preferred nullability context in the generated files, for example by adding `#nullable enable`.

## Disable StyleCop Analyzers in generated code

Remember to add `// <auto-generated />` comment at the top of all the generated files, otherwise StyleCop Analyzers could emit warnings in the generated code. 

# Debugging

## Create debugging launch profile

Debugging used to be a really painful experience, but fortunately the situation greatly improved from [Visual Studio v16.10](https://learn.microsoft.com/en-us/visualstudio/releases/2019/release-notes-v16.10#NETProductivity) that introduced first class debugger support for source generators. 

In order to use the debugger support you need to add the `<IsRoslynComponent>true</IsRoslynComponent>` to the project file of the source generator project and be sure to have the `.NET Compiler Platform SDK` component installed. 

At this point my suggestion would be to create a project that can become your *playground* for debugging (and testing) the source generator. After creating the playground project, you should add a reference to your source generator project and edit the `csproj` file like this:

```xml
<ItemGroup>
    <ProjectReference Include="pathToSourceGeneratorProject" OutputItemType="Analyzer" ReferenceOutputAssembly="false" />
</ItemGroup>
```

Then you can create a `Roslyn Component` debug launch profile from the `Debug` section of properties the source generator project. In the profile you can specify the target project, that is one of the project that is referencing the source generator as showing before. Running this launch profile will execute the source generator and you will able to debug it as any other project.

This kind of playground project is also particularly useful to have a first verification that eventual diagnostics have been generated correctly, and at the right position in the user code. 


## Persist generated files

In order to see the generated files from the source generator you can check under `Dependencies -> Analyzers -> SourceGeneratorProjectName -> SourceGeneratorName`. Even though all the generated files are there, there are some issues regarding [the caching of the source generator when referencing it locally](https://github.com/dotnet/roslyn/issues/48083). This caching behavior results in, sometimes, not being able to see any change in the generated files, even though there have been changes in the source generator itself. In these cases the best option is just to restart Visual Studio. 

There is actually a better way to have a look at the generated files that does not suffer from those issues. What you can do is to persist the generated files to disk by modifying the project file:

```xml
<Project Sdk="Microsoft.NET.Sdk">
	<PropertyGroup>
		<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
		<CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
	</PropertyGroup>
	<ItemGroup>
		<ProjectReference Include="pathToSourceGeneratorProject" OutputItemType="Analyzer" ReferenceOutputAssembly="false" />

		<!-- Removing the generated files from compilation, adding them back as non-content files -->
		<Compile Remove="$(CompilerGeneratedFilesOutputPath)/**/*.cs" />
		<None Include="$(CompilerGeneratedFilesOutputPath)/**/*.cs" />
	</ItemGroup>
</Project>
```

`EmitCompilerGenerated` files set to true will emit the generated files to disk, while `CompilerGeneratedFilesOutputPath` specifies the folder in which the files need to be saved. Because the generated files have been already added to the compilation by the source generator, you need to remove the emitted files from the compilation, and you can do that with the `<Compile Remove=...>` tag. Finally, if you want to be able to see the generated files in the Visual Studio project, you can add them back with the `<None Include=...>` tag, that represents files that are ignored the build process. 

Emitting the generated files not only allows to have direct feedback on the source generator, but it also makes possible to add those files to source control. In my opinion that is particularly useful, as it allows to see what kind of effect certain changes of the source generator code have on the generated files.

# Testing

## Unit testing

When developing a source generator there are essentially two things that can be tested: how the generated code behaves, and how the generated code looks like. We will focus on the second part in this section, as the first is no different than testing any other code. 

In order to test that the generated files are generated correctly, I suggest to follow what is described in the `Unit Testing of Generators` section of the [Roslyn Cookbook](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md#unit-testing-of-generators). Personally I prefer the first solution to testing, that makes use of the *verifiers* contained in the ` Microsoft.CodeAnalysis.Testing` packages, so that the source generator can be tested in a similar way to an analyzer. 

I am not going to go into details about how to use these, as it's shown in the cookbook, but the main idea here is to specify the source code, the generated code, eventually what kind of diagnostics should be emitted, and the verifier takes care of running the source generator and check that the results are as expected. 

Depending on the kind of source generator that you will be working on, this process of specifying text strings for the source and generated files, as well as the diagnostics could be quite cumbersome. For this reason I have decided to use a more "flexible" approach for the unit testing of the source generator included with the Realm SDK:
- I have created a *playground* project that references the source generator project as described in the previous section. This project will contain all the source files that are used for unit testing, including files that should have diagnostics.
- The source generated files are emitted to the file system as explained in the previous section
- Additionally, the source generator is also generating files containing eventual debugging diagnostics that need to be emitted from the test files. The diagnostics are just serialized in a file as JSON
- The verifier reads the source files, the generated files as well as the diagnostics directly from the file system for testing

This approach relies on verifying manually that the generated files are as we expect them to be, and it could be argued that this is a naive way of testing, but in my opinion this is *good enough*. In the end these tests are not verifying that the generated code works, but just confirm that it looks as expected, so we do not need to be unnecessarily strict from my point of view.

I think that this approach is also particularly useful in verifying that eventual diagnostics are generated correctly, as it is possible to confirm visually that the diagnostics are not only correct, but also in the right position in the code. If we had to specify the diagnostic objects in code with the verifier, then it would be much more difficult to understand their correctness, especially regarding positioning.

## Various

The Roslyn cookbook shows a basic example of how to do source generator unit testing. In the following part I have gathered some small tips on how to modify the verifier to accommodate some more complex needs.

### Disable compiler diagnostics

By default, the verifiers validates all kind of diagnostics coming from the source code. It could be convenient to validate only the diagnostics emitted from the source generator. This can be done like this:

```csharp
public class Test : CSharpSourceGeneratorTest<TSourceGenerator, NUnitVerifier>
{
    public Test()
    {
        // Removes the emission of the usual compiler diagnostics
        CompilerDiagnostics = CompilerDiagnostics.None;
    }
    //....
}
```

## Add reference to framework assemblies

By default the verifier uses default reference assemblies depending on the target framework of the test project. You could be wanting to change the default reference assemblies depending on the source generator. This can be done like this:

```csharp
public class Test : CSharpSourceGeneratorTest<TSourceGenerator, NUnitVerifier>
{
    public Test()
    {
        // This uses the .NET 6.0 reference assemblies, for instance
        ReferenceAssemblies = ReferenceAssemblies.Net.Net60;
    }
    //....
}
```

## Add reference to your assemblies

You may need to add additional references to your verifier. This can be done like this:

```csharp
public class Test : CSharpSourceGeneratorTest<TSourceGenerator, NUnitVerifier>
{
    public Test()
    {
        TestState.AdditionalReferences.Add(typeof(YourClass).Assembly.Location);
    }
    //....
}
```

## Diagnostic testing


# Others

Suppress warnings on generated code (put pragmas around file) -- Auto - generated?

