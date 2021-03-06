# Type inheritance support in Swagger API

Thanks to [PR #2063](https://github.com/VirtoCommerce/vc-platform/pull/2063), Virtocommerce platform is able to expose derived types in Swagger API description. 

## Problem
Consider the following code:
```csharp
// Models definition in ...Core assembly

public abstract class BaseObject
{
    // ...
}

public class DerivedObject : BaseObject
{
    // ...
}

public class AnotherDerivedObject : BaseObject
{
    // ...
}

// Controller method that returns these models
public ActionResult<IList<BaseObject>> GetObjects()
{
    var result = new[] 
    {
        new DerivedObject(),
        new AnotherDerivedObject()
    };

    return Ok(result);
}
```

In this example, only the `BaseObject` will be exposed in Swagger. `DerivedObject` and `AnotherDerivedObject` will be missing from JSON with API definition, and the only way to expose them is to add some other API method that will explicitly accept or return any of these derived types.

On the other hand, we don't want to enable polymorphism globally, as this might break existing API (e.g. any action from `vc-module-customer` that works with a `Member` class).

## Solution
Since the creation of VC platform v3, polymorphism support in Swashbuckle [improved significantly](https://github.com/domaindrivendev/Swashbuckle.AspNetCore/pull/1792). Now we actually can use it in action, and the resulting document will be suitable for AutoRest (e.g. to generate API clients for storefront). An example mentioned above can be reworked like this to expose derived models:

```csharp
using Swashbuckle.AspNetCore.Annotations;

[SwaggerSubType(typeof(DerivedObject)]
[SwaggerSubType(typeof(AnotherDerivedObject)]
public abstract class BaseObject
{
    // ...
}

public class DerivedObject : BaseObject
{
    // ...
}

public class AnotherDerivedObject : BaseObject
{
    // ...
}
```

This will expose `BaseObject`, `DerivedObject` and `AnotherDerivedObject` in Swagger API description (despite the fact that `GetObjects()` method still has only the base type in its signature), and it won't break other API.

More info on used attributes and type descriminator annotations could be found [here](https://github.com/domaindrivendev/Swashbuckle.AspNetCore#enrich-polymorphic-base-classes-with-discriminator-metadata).

## Limitations
This approach works and does not break any existing clients generated by AutoRest. However, it has some limitations:
1. If `BaseObject` has any abstract or virtual properties that are overridden in derived types, Swashbuckle will include these properties both to `BaseObject` and to derived types. However, `AutoRest` does not understand that and produces an error like `FATAL: System.InvalidOperationException: Found incompatible property types ,  for property 'someVirtualProperty' in schema inheritance chain`. A solution for this would be to avoid using abstract and virtual properties for such types.
2. This approach only works if all of derived types are located in the same module - it won't allow to extend this list from other modules. To overcome this, we might need to make a custom sub-type selector - its code would be based on the [existing implementation in Swashbuckle](https://github.com/domaindrivendev/Swashbuckle.AspNetCore/blob/master/src/Swashbuckle.AspNetCore.Annotations/AnnotationsSwaggerGenOptionsExtensions.cs#L70-L90), but use `AbstractTypeFactory` instead of custom attributes to find descendant types. However, this might require extending the `TypeInfo`, so that we could explicitly specify what types can be exposed in Swagger API.
