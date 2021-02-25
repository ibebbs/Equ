
[![Build Status](https://travis-ci.org/thedmi/Equ.svg?branch=master)](https://travis-ci.org/thedmi/Equ)
[![NuGet](https://img.shields.io/nuget/v/Equ.svg)](https://www.nuget.org/packages/Equ/)


Equ
====

Equ is a *.NET Standard* library that provides fast, convention-based equality operations for your types. This is primarily useful for custom value types (in the sense of the [value object pattern](http://en.wikipedia.org/wiki/Value_object), not in the sense of `struct`). This is similar to C# 9 records, but extends the concept to enumerable members.

Members participating in equality operations are resolved through reflection, the resulting equality operations are then stored as compiled expression using LINQ expression trees. This makes the runtime performance as fast as with a manual implementation.

Compared to C# 9 records, this library adds the following functionality:

- Compare sequences by value, too.
- Exclude members from comparison through attributes.
- Extend comparison without having to re-implement boiler plate.

Grab the NuGet [here](https://www.nuget.org/packages/Equ/).


Usage
----------


### Simple Scenarios

The easiest way to leverage Equ is to derive your class from `MemberwiseEquatable<TSelf>` where `TSelf` is the deriving class itself. `MemberwiseEquatable<TSelf>` uses field-based equality, so your class gets `Equals()` and `GetHashCode()` implementations that consider all fields.

#### Example

The following example shows a very simple value object called `Person`:

```csharp
using Equ;

class Person : MemberwiseEquatable<Person>
{
    public Person(string name, Address address)
    {
        Name = name;
        Address = address;
    }

    public string Name { get; }
    public Address Address { get; }
}

class Address : MemberwiseEquatable<Address>
{
    public Address(string street, string city)
    {
        Street = street;
        City = city;
    }

    public string Street { get; }
    public string City { get; }
}
```

With these two value objects, the following expression is true:

```csharp
new Person("Sherlock", new Address("Baker Street", "London")) == new Person("Sherlock", new Address("Baker Street", "London")) // true
```

This works because `MemberwiseEquatable` provides an overload for the `==` operator that eventually *compares all private fields* of the objects in question (note that auto properties have compiler-generated backing fields).


#### With C# 9 Records

Since records already provides the basis for value types, it makes sense to use them if C# 9 is available in your project. However, C# 9 records are problematic when enumerable members are used, because they perform a reference comparison in that case. This breaks compositional integrity, but Equ can fix this. However, records cannot inherit from `MemberwiseEquatable` as in the example above, so there is a different way:

```csharp
record Person(string Name, IReadOnlyList<Address> Addresses)
{
    // Redefine Equals() and GetHashCode() so that Equ is used
    public virtual bool Equals(Person other) => EquCompare<Person>.Equals(this, other);
    public override int GetHashCode() => EquCompare<Person>.GetHashCode(this);
}

record Address(string Street, string City) { }
```

Now the following expression evaluates to true, which it wouldn't with plain records:

```csharp
new Person("Sherlock", new [] { new Address("Baker Street", "London") }) == new Person("Sherlock", new [] { new Address("Baker Street", "London") })
```


### Excluding Members from Equality Comparison

To exclude certain fields (or properties if using property-based comparison) from equality comparison, just mark them with the `[MemberwiseEqualityIgnore]` attribute like so:

```csharp
using Equ;

class Record : MemberwiseEquatable<Record>
{
    public Record(string value1, string value2, string transientValue)
    {
        Value1 = value1;
        Value2 = value2;
        TransientValue = transientValue;
    }

    public string Value1 { get; }

    public string Value2 { get; }

    [MemberwiseEqualityIgnore]
    public string TransientValue { get; }
}

// later...

new Record("v1", "v2", "A") == new Record("v1", "v2", "B") // true
```

### Customizable Equality Comparer

For most scenarios, the simple usage should be fine. But if you need more control or do not want to inherit from `MemberwiseEquatable<TSelf>`, just implement `IEquatable<T>` and delegate `Equals()` and `GetHashCode()` to an instance of `MemberwiseEqualityComparer<T>`. `MemberwiseEqualityComparer<T>` offers static instances through `ByFields` and `ByProperties`.

For very advanced scenarios, you can even create a completely customized comparer by using `MemberwiseEqualityComparer<T>.Create(EqualityFunctionGenerator)`.

#### Example

The same example as above, only this time we don't inherit from `MemberwiseEquatable`.

```csharp
using Equ;

class Address : IEquatable<Address>
{
    // Make sure the comparer is static, so that the equality operations are only generated once
    private static readonly MemberwiseEqualityComparer<Address> _comparer =
        MemberwiseEqualityComparer<Address>.ByFields;

    public Address(string street, string city)
    {
        Street = street;
        City = city;
    }

    public string Street { get; }
    public string City { get; }

    public bool Equals(Address other)
    {
        return _comparer.Equals(this, other);
    }

    public override bool Equals(object obj)
    {
        return Equals(obj as Address);
    }

    public override int GetHashCode()
    {
        return _comparer.GetHashCode(this);
    }
}
```

### Compositional Integrity

At the core of Equ lives the `EqualityFunctionGenerator`, which is responsible for generating `Equals()` and `GetHashCode()` expressions given a type and a set of `MemberInfo` instances. The generator maintains compositional integrity in the sense of value objects, i.e. it compares objects value by value. To fully leverage this concept, the generator follows a set of rules when using members in an equality operation:

- For reference-typed members, a call to their `Equals()` or `GetHashCode()` methods is generated
- For value-typed members an equals operation (`a == b`) is generated, or the `GetHashCode()` of the boxed type is called
- For sequence-typed members, a call to an appropriate `ElementwiseSequenceEqualityComparer<T>` is generated

The `ElementwiseSequenceEqualityComparer<T>` is basically just a wrapper around `Enumerable.SequenceEqual()` with additional null checks.

C# 9 records do the same, except that they don't consider sequence-typed members.

## Release Notes

See [Release Notes](ReleaseNotes.md).
