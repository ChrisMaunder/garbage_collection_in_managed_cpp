# Garbage Collection in .NET

A quick introduction to Garbage collection in .NET using Managed C++



## Background

The inclusion of Garbage Collection in the .NET runtime removes the need to track and release memory allocations. When programming in the managed environment you allocate memory on the managed heap using the new operator, but instead of deleting or freeing that memory, you simply remove all references to that memory location (eg. by setting your pointer to that memory to `NULL`) and let the garbage collector (GC) take care of the rest. Note that we are talking only about memory - not resources. If you create a new object on the managed heap and that object allocates resources such as handles or connections then you must ensure that that object has released it's resources before casting the element adrift to the mercy of the GC. 

The traditional C runtime heap is essentially a linked list that required traversing each time a memory allocation was made. Once a suitable block of memory is located this block would be split and the memory location returned. In contrast, the managed heap in .NET contains a single pointer that always points to the next available spot in memory, and the pointer marking the top of the heap is moved accordingly when an object is allocated. There is no traversal or splitting necessary, which means allocating memory on the managed heap is almost as fast as allocating memory on the stack. 

This memory allocation model works perfectly if one assumes you have infinite memory - which invariably you won't. So what happens when you attempt to allocate a block of memory at the end of the managed heap but you find that you have run out of memory? Garbage collection, of course. In a typical application you would normally be allocating and deallocating memory through the course of your application's logic. In a managed application this deallocation will not happen, but rather the memory that is no longer in use will sit there, unreferenced, until you get to the point where there is no more free memory and the GC is forced to reclaim all this unused memory. 

## Garbage Collection 

The process of garbage collection starts with the GC assuming all memory on the managed heap is rubbish. It then lists all your application's global and static memory pointers, local or parameter variables and CPU registers containing references to objects on the heap, and then uses these objects to build a graph of all objects on the heap that are either directly or indirectly referenced by your application. Any object on the managed heap that can somehow be accessed by your application will be marked. There are optimisations in place to remove the possibility of circular memory references causing infinite loops, and to ensure that chains of references are only processed once.

Once the graph is complete the GC now has a complete picture of what is, and isn't, garbage. The GC then compacts the heap by moving non-garbage items together and then resets it 'Next available memory slot' pointer to the top of this new, compacted heap. In doing so the GC is also responsible for updating the values of all pointers into the heap, so that references to objects on the heap that have been moved are still valid.

Large objects (&gt; 85,000 bytes) are treated a little different that smaller objects. Objects of this size are allocated on a separate large heap and when garbage collection occurs these objects are not moved around since moving blocks of memory this large can really start to slow things down. 

If, after garbage collection has occurred, there is still insufficient memory for the memory allocation request, an `OutOfMemoryException` exception is thrown.

## Generations

While memory allocation on the managed heap is fast, GC itself may take some time. With this in mind several optimisations have been made to improve performance. The GC supports the concept of generations, based on the assumption that the longer an object has been on the heap, the longer it will probably stay there. When an object is allocated on the heap it belongs in generation 0. Each garbage collection that that object survives increases its generation by 1 (currently the highest supported generation is 2). Obviously it's faster to search through, and garbage collect a subset of all objects on the heap, so the GC has the option of collecting only generation 0, 1 or 2 objects (or whatever combination it chooses until it has sufficient memory). Even while collecting only younger objects the GC can also determine if old objects have references to new objects to ensure that it doesn't inadvertently ignore in-use objects.

`System.GC` contains the garbage collection object, and there are a number of static methods you can use to have direct control over the process. 

`void GC::Collect()` invokes the GC for all generations, while `void GC::Collect(int Generation)` invokes it only up to and including the generation you specify. 

The maximum supported generation can be found by querying `GC::MaxGeneration`, and if you are curious as to what generation your object currently resides in you can call `int GC::GetGeneration(Object* obj)`. 

You would normally leave the GC alone to do its own thing, but if you are aware that your application is about to start needing huge chunks of memory quickly, or you know that you have finished using a ton of memory and you are having a little idle time, then it may be worthwhile to give the GC a hint that its services are needed. 

If you are ever curious as to the number of bytes currently thought to be allocated, just call 

```cpp
_int64 GC::GetTotalMemory(bool forceFullCollection);
```

The parameter *forceFullCollection* determines whether or not the function should wait for a GC to occur before reporting the amount of memory. 

## Weak References

You may need to allocate an object on memory, but then only need to reference that object occasionally. It would be nice if you could keep track of that object, but release it to the GC if memory started getting a little tight. This can be achieved using Weak References. Essentially you allocate an object, create a weak reference to that object and then remove any direct references you have to that object so the GC can claim it if it needs to. 

Weak references are created using the `WeakReference` class, and then accessed by recreating a strong reference to the object by accessing `WeakReference::Target`. Once you are done with the object you can then again release it as a weak reference. 

```cpp
// Create your managed object
MyObject* myObj = new MyObject();

// Create a weak reference
WeakReference* weak = new WeakReference(myObj);

// Remove your direct (strong) reference
myObj = 0;

... 

// Now do all accessing of your object via the weak reference
object  myObj = weak->Target; 
if (myObj)
{ 
    // The object still exists - We can use it!
    ... 
    // Release all strong references again. Use weak->Target to grab it again later.
    myObj = 0;
} 
else
{
    // The object is shark bait. Time to create another and use it instead :(
}    
```

The other option in checking if a weak reference is still valid is to query the `WeakReference::IsAlive` property. 

So how do you maintain a reference to an object (even a weak reference) and still allow the GC to collect it if the GC is invoked? When a weak reference is made, an entry into a weak reference table is made which holds the location of the object (this is what your object pointer's value will be). This table is not taken into account when the GC walks the applications objects, but after the GC makes a list of all garbage objects, it looks up the weak reference table for all entries that point to garbage. These entries are set to NULL, and the objects reclaimed, and so the next time you try and access the object via the weak reference you will get back NULL. 

## Memory Allocation

One important point with memory allocation on the managed heap is that when you allocate successive memory blocks you are assured that you will receive successive memory locations. By having related objects physically close to each you get the advantage of fewer page faults and a higher likelihood that the objects will be in the cache. This is a nice performance booster.

To actually declare and create an object that is accessible by the garbage collector you need to prefix your class declaration by the `__gc` keyword. For example: 

```cpp
__gc class CMyGCClass
{
    int n;
};
```

This will declare a class `CMyGCClass` and mark it as managed, meaning that it must be declared on the managed heap, and will be subject to garbage collection. To create the object you need to create it on the heap using the `new` keyword: 

```cpp
CMyGCClass *gc = new MyGCClass();
```

To release the object simply set its reference to NULL 

```cpp
gc = 0;
```

## Finalizers and Dispose, and resource management

The base .NET object `System::Object` contains an overridable method `Finalize` that is automatically called by the framework in order to give the object a chance to release all resources before it's destroyed. The signature is: 

```cpp
protected: virtual void Object::Finalize();
```

In beta versions of the .NET Framework one was able to override this function, but this is no longer the case in the final release of .NET for Managed C++ and C#. Instead you use the standard destructor syntax. This is both a convenience and a safety net in that it means less typing and gives a guarantee that your base class implementation of `Finalize` will be called. Behind the scenes when you write 

```cpp
~CMyGCClass()
{
   // Cleanup code
}
```

The compiler translates this to

```cpp
protected:
    void Finalize()
    {
       try
       {
          // Cleanup code
       }
       finally
       {
          CMyGCBaseClass::Finalize();
       }
    }
```

Your destructor, and hence `Finalize` method will be called by the garbage collector at an undetermined time after the object undergoes garbage collection. You can't rely on it being called at the time you release your reference to the object. This means that resources that your object may have allocated (eg Handles or database connections) may be held until after the program ends (unless you call `GC::Collect()`, which is using a rather large hammer to solve a simple problem). Implementing a finalization method also promotes the object to a higher generation, meaning that they may stay in memory longer than necessary. Finalization is also not guaranteed to occur for objects that are still alive when the application exits (this allows the application to shutdown faster). Resources will be reclaimed, but not gracefully. Also, finalizable objects take longer to allocate. 

Instead of implementing a `Finalize` method you are encouraged to inherit your classes from the `IDisposable` interface implement a `Dispose` method. 

```cpp
public virtual void Dispose();
```

This method should take care of all resource deallocation and also call `GC:SuppressFinalization(this)` which instructs the runtime not to call `Finalize` on your object since you have already done all cleanup. There is a cost associated with having `Finalize` called, so if you can avoid it, do so. 

The recommended pattern is to declare a `Dispose` method (and inherit from `IDisposable`) and call this when you want to free your object's resources. 

```cpp
CMyClass *mc = new CMyClass();
mc->DoSomething();
mc->Dispose();
mc = 0;
```

The above code snippet allocates an object, uses the object (possible allocating resources along the way) then forces the object to clean up in a deterministic way before being released to the whims of the GC. You have complete control over all resource management, and leave the managed heap management to the GC. 

Implementing such a class is done as follows: 

```cpp
__gc class CMyClass: public Object, public IDisposable
{
public:
    CMyClass()  
    {
        m_bDisposed = false;
    }

public:
    // You are encouraged to not allow a disposed object to be reused
    void MyMethod()
    {
       if (!m_bDisposed)
       {
           // do something
       }
       else
       {
           // throw an exception
       }
    }

public:
    // In case you would like to call 'Close' instead of 'Dispose'
    void Close() 
    {
        Dispose();
    }

    // defined in IDisposable
    void Dispose()
    {
        if (!m_bDisposed)
        {
            m_bDisposed = true;

            // Free all resources

            GC::SuppressFinalize(this);  
        }
    }

protected:
    bool m_bDisposed;
};
```

You are strongly encourage to disallow the use of an object once it has been Disposed. Note in the above code that we have provided both a `Finalize` method which will be called automatically, and a `Dispose` method that must either be called by the user, or called via a containing object. In the `Dispose` method there is the line 

```cpp
GC::SuppressFinalize(this); 
```

This instructs the GC to **not** call the `Finalize` method. We have already cleaned up and so don't want to incur the overhead of a call to `Finalize`. 

## Example Code

Below is a simple code snippet that demonstrates some of the main points of garbage collection: 

```cpp
Console::WriteLine("Garbage Collection demonstration\n");

__int64 TotalMem = GC::GetTotalMemory(true);
Console::WriteLine("Currently used {0} bytes of memory", TotalMem.ToString());

// Wanton disregard for all the things we've been taught
Console::WriteLine("Creating a ton of junk");
for (int i = 0; i < 10000; i++)
    new MyGCClass();

TotalMem = GC::GetTotalMemory(false);
Console::WriteLine("Have now used {0} bytes of memory", TotalMem.ToString());

GC::Collect();

TotalMem = GC::GetTotalMemory(false);
Console::WriteLine("After GC, used {0} bytes of memory", TotalMem.ToString());

// -------------------------------------------------------------
// Demonstration of Generations...

Console::WriteLine("\nDemonstration of Generations\n");

String* str = new String("This is a string");
Console::WriteLine("We have created the string '{0}'", str);

// How old is it?
int nMaxGen = GC::MaxGeneration;
int nGen = GC::GetGeneration(str);
Console::WriteLine("The object's generation is '{0} (max {0})'", 
    nGen.ToString(), nMaxGen.ToString());

// Let's make it older and wiser
Console::WriteLine("Garbage Collecting...");
GC::Collect();
nGen = GC::GetGeneration(str);
Console::WriteLine("The object's generation is '{0} (max {0})'", 
    nGen.ToString(), nMaxGen.ToString());

Console::WriteLine("Garbage Collecting...");
GC::Collect();
nGen = GC::GetGeneration(str);
Console::WriteLine("The object's generation is '{0} (max {0})'", 
    nGen.ToString(), nMaxGen.ToString());

Console::WriteLine("Garbage Collecting...");
GC::Collect();
nGen = GC::GetGeneration(str);
Console::WriteLine("The object's generation is '{0} (max {0})'", 
    nGen.ToString(), nMaxGen.ToString());

// -------------------------------------------------------------
// Demonstration of Weak References...

Console::WriteLine("\nDemonstration of Weak References\n");

Console::WriteLine("Creating a weak reference.");
WeakReference* weak = new WeakReference(str);
str = 0;

if (weak->IsAlive)
{
    str = (String*) weak->Target;
    Console::WriteLine("Object is alive. [{0}]", str);
    str = 0;
}
else
    Console::WriteLine("Object is gone");

GC::Collect(0);

if (weak->IsAlive)
{
    str = (String*) weak->Target;
    Console::WriteLine("Object survived GC of generation 0. [{0}]", str);
    str = 0;
}
else
    Console::WriteLine("Object is gone (collected with Gen 0)");

GC::Collect(1);

if (weak->IsAlive)
{
    str = (String*) weak->Target;
    Console::WriteLine("Object survived GC of generation 1. [{0}]", str);
    str = 0;
}
else
    Console::WriteLine("Object is gone (collected with Gen 1)");

GC::Collect(2);

if (weak->IsAlive)
{
    str = (String*) weak->Target;
    Console::WriteLine("Object survived GC of generation 2. [{0}]", str);
    str = 0;
}
else
    Console::WriteLine("Object is gone (collected with Gen 2)");

// -------------------------------------------------------------
// Demonstration of Finalize/Dispose...

Console::WriteLine("\nDemonstration of Finalize/Dispose\n");

Console::WriteLine("Creating a CFinalizeTest object.");
CFinalizeTest* ft = new CFinalizeTest();
Console::WriteLine("Calling 'Close' then allowing the GC to collect it.");
ft->Close(); // or ft->Dispose() if you want
ft = 0;
GC::Collect();

Console::WriteLine("Creating a CFinalizeTest object, and will now delete it");
ft = new CFinalizeTest();
delete ft;
ft = 0;

Console::WriteLine("Creating a CFinalizeTest object, no Close or release to the GC.");
ft = new CFinalizeTest();

Console::WriteLine(" -- PROGRAM ENDS --\nPress Enter to continue");
Console::ReadLine();
```

## History

26 Apr 2001 - added demo application, updated source code sample. 

16 Oct 2001 - updated for beta 2. 

18 Jun 2002 - updated for RTM.
