<!--Auto-generated from .cs file, the edits here will be lost! -->

# Thread Safety


- [Thread Safety](#thread-safety)
  - [DryIoc is Thread-Safe](#dryioc-is-thread-safe)
  - [Locking is only used for reused service creation](#locking-is-only-used-for-reused-service-creation)


## DryIoc is Thread-Safe

DryIoc ensures that:

- Registrations and resolutions could be invoked concurrently on the same container (possibly from a different threads) without corrupting container state.
- Registrations do not stop-the-world for resolutions. __There is no locking associated with registrations__.
- Resolutions do not block registrations either. __No locking except for reused instances as explained below__.
- Registration won't be visible for resolutions until it is completed. If it is incomplete (due exception) all its changes will be ignored.

The above guaranties are possible because of the Container data-structure. 
In a pseudo code and simplifying things a lot the DryIoc Container may be represented as following:
```cs 
//md{ usings ...
using System;
using System.Linq.Expressions;
using System.Threading.Tasks;
using NUnit.Framework;
using ImTools;
using DryIoc;
#pragma warning disable CS0649
// ReSharper disable UnusedVariable
//md}

class Oversimplified_container 
{
    class Ref<T> { public T Value; } // represents the CAS (compare-and-swap) box for the referenced value

    class Container 
    { 
        public Ref<Registry> Registry;
    }

    class Registry 
    {
        public ImHashMap<object, Expression<Func<Container, object>>> Registrations;
        public Ref<ResolutionCache> ResolutionCache;
    }

    class ResolutionCache 
    {
        public ImHashMap<object, Func<Container, object>> Cache;
    }
}
```

Given this structure, Registration of new service will produce new `registry`, then will swap old `registry` with new one inside the container. If some concurrent registration will intervene then the Registration will be retried with new `registry`.

Resolution on the other hand will produce new `resolutionCache` and will swap new cache inside the same registry object. This ensures that resolution cache is bound to the specific registry, and does no affected by new registrations (which produce new registries). 

__Note:__ In addition to providing thread-safety the described Container structure allows to produce new Container very fast, at O(1) cost. This makes creation of ["Child" containers and Open Scope](KindsOfChildContainer) a cheap operation.


## Locking is only used for reused service creation

Lock is required to ensure that creation of reused service happens only once. For instance for singleton service A:
```cs 
class Resolving_singleton_in_parallel 
{
    [Test] public void Example() 
    {
        var container = new Container();

        container.Register<A>(Reuse.Singleton);

        Task.WaitAll(
            Task.Run(() => container.Resolve<A>()),
            Task.Run(() => container.Resolve<A>()),
            Task.Run(() => container.Resolve<A>())
        );

        Assert.AreEqual(1, A.InstanceCount);
    }

    public class A 
    {
        public static int InstanceCount;

        public A() { ++InstanceCount; }
    }
}
```

The lock boundaries are minimized to cover service creation only - the rest of resolution bits are handled outside of lock. 
