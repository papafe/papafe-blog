---
title: "A new post"
date: 2022-06-01T22:52:34+02:00
draft: false
---

## one

When creating an object is common to use the object initializer syntax to initialize properties:

```csharp
var a = new MyClass
{
    Val1 = 23,
    ...
}
```

## two


This is creating the object and calling the setter on the created object to assign the properties. 
This is different if one is trying to use it with collections and class (?) properties 

```csharp
var a = new MyClass
{
    Collection =  { 1, 2, 3, 4},
    InnerClass = 
    {
        InnerVal = 23
    }
    ...
}
```

## three-one
For the collection, what is happening is that it's adding those values to the collection.
For the class, it's calling the setters on those properties. 
In both cases this is not creating anything (either a collection or a new object) so those must have been already initialized/created by the class.
This also means that this kind of syntax can be used also with getter only collection/class properties, because it doesn't set them, but only modifies them.


Regarding collections, it's enough that implements IEnumerable and has an Add method (probably duck typing). This means that we can create all the customized add methods that we want, not only the ones implemented by the base collection. 
And the values passed in the initialized are passed directly to Add, so it's very customizable




https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/object-and-collection-initializers