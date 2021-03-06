---
layout: post
title: C++ does have automatic memory management.
date: '2012-08-01'
categories:
  - Software
comments: true
---

Some weeks ago someone was discussing what language to use to write a small library. He wanted to go with C. I suggested using C++ for the implementation but keeping a pure C API is an interesting alternative. He said C++ added complexity but not benefits like Go would do with automatic memory management, and in that case C would be the best option.

This is a common miss-understanding. Modern C++ **does have** automatic memory management. It does not have a Garbage Collector, which is **one form** of automatic memory management.

Consider the following:

```cpp
void someFunction()
{
    SomeClass \*obj = new SomeClass();
    ...
    delete obj;
}
```

Here you have to take care that _obj_ gets deleted before leaving the function. When the function ends, only the pointer is destroyed, because is a local variable allocated on the stack, but no the object the pointer is pointing to.

The problem begins with exceptions. If the code between the allocation and the _delete_ throws an exception, _delete_ is not called and the object is not deleted. Of course you can _catch_ the exception, _delete_ the object and then rethrow the exception. Not very nice.

## Enter RAII

RAII is a name I dislike. Stands for "Resource Acquisition Is Initialization", but is a powerful concept in C++: The language guarantees that the destructor gets called for an object that is allocated on the stack when it goes out of scope.

```cpp
// global mutex
Mutex mutex;

void someFunction()
{
    mutex.lock();
    ...
    mutex.unlock();
}
```

Here we have the same problem. If an exception is thrown, the mutex is never unlocked!. Lets create a solution based on the concept of RAII.

```cpp
class Lock
{
    Lock(Mutex *m)
     : _mutex(m)
    {
        _mutex->lock();
    }

    ~Lock()
    {
        _mutex->unlock();
    }
private:
    Mutex *_mutex;
};
```

And now incorporate it into the code:

```cpp
void someFunction()
{
    Lock lock(&mutex);
    ...
}
```

As soon as the lock is constructed, it locks the mutex. If an exception is thrown or the function ends, _lock_ goes out of scope and it destructor is called. The destructor calls _unlock()_ on the mutex, which we stored a pointer when constructing the lock. The lock itself does not need to be deleted as it is a local variable on the stack.

Now imagine you want to extend this to various classes that can provide lock() and unlock(). You would create a template for the class.

So how can this concept be used for memory management. Lets take the _Lock_ and implement a _Deleter_ from it. We will use a template:

```cpp
template<class T> class Deleter
{
    Deleter(T *ptr)
     : _ptr(ptr)
    {
    }

    ~Deleter()
    {
        delete _ptr;
    }
private:
    T *_ptr;
};
```

See? it is the same as the _Locker_ but not lock() call, and instead of unlock() we delete the object.

Now, you may ask yourself, what is the usefulness of having the pointer wrapped into another object. True. Lets provide access to it. Thanks to C++ operator overloading, we can overload the -\> operator which is the one we already use when working with pointers.

```cpp
template<class T> class Deleter
{
    // ...
    // ...

    T* operator-> ()
    {
        return _ptr;
    }

private:
    T *_ptr;
};
```

Now we can access the underlying pointer in a natural way. Exactly the same as we would use it if it was naked and not wrapped in this smart deleter:

```cpp
void someFunction()
{
    Deleter<SomeClass> ptr(new SomeClass());
    // hello() is a method of SomeClass
    ptr->hello();
    ...
}
```

When we call '_ptr->hello()' the '->' operator of the Deleter returns a pointer to the member object, so we can directly call methods on it.

Now if an exception is thrown or the function returns, 'ptr' goes out of scope. _Deleter_'s destructor is called and the object is deleted.

This is called a smart pointer. This implementation is the most basic one. If you try to return it from the function it will not work as it would create a copy of the Deleter, pointing to the same underlying pointer and it would delete the object twice. But from here you can extend the concept by implementing copy constructors that add reference counting and much more.

## This is all now built-in

Originally, you could get smart pointer implementations from the [boost smart pointer library](http://www.boost.org/doc/libs/1_50_0/libs/smart_ptr/smart_ptr.htm). However C++11 already incorporates most of them in the standard library.

### auto_ptr and unique_ptr

The basic implementation I showed you before is provided by the standard library as std::auto_ptr. With the only difference is that auto_ptr is more robust and if it is copied, the original one loses the pointer (gets changed to 0) in order to avoid double deletion:

```cpp
#include <iostream>
#include <memory>
using namespace std;

int main(int argc, char **argv)
{
    SomeClass *c = new SomeClass();
    auto_ptr<SomeClass> x(c);
    auto_ptr<SomeClass> y;

    y = x;

    cout << x.get() << endl; // this one is 0
    cout << y.get() << endl; // this one is some memory address

    return 0;
}
```

In C++11 auto+ptr was deprecated (but still available) and replaced to std::unique_ptr which can't be copied, but the pointer can be transferred.

```cpp
std::unique_ptr<SomeClass> p1(new SomeClass());
std::unique_ptr<SomeClass> p2 = p1; // this will not compile
std::unique_ptr<SomeClass> p3 = std::move(p1); // You can manually transfer it though, and p1 will be set to 0
```

### shared_ptr

std::shared_ptr​ adds reference counting, so if you copy it, it points to the same pointer but the reference count gets increased. When the last one gets destructed the pointer is deleted.

```cpp
std::shared_ptr<SomeClass> p1(new SomeClass());
std::shared_ptr<SomeClass> p2 = p1;

// the pointer will be deleted when both go out of scope and the reference count goes to 0
```

The problem with reference counting is that if two objects reference each other, then the counts never goes to zero. What you do in this case is that one has shared_ptr and the other uses a weak_ptr. A weak_ptr does not increase the reference count of the shared_ptr. When you want to access the underlying object, you can obtain a shared_ptr from the weak_ptr. But you will need to test the pointer first, as it may have been deleted.

## Conclusion

C++ does provide automatic memory management. It requires more thinking than a garbage collector but it is more deterministic and it is built on principles that can be used for general resource management like files, locks, temporary files, etc.

In addition to that you get real strings, containers, etc. plus full C interoperability. You can keep the complexity away from your programs choosing carefully what idioms and concepts your program will use. The complexity can stay for library developers (e.g. boost) who need to use all C++ features in order to provide flexible components.

## Nothing new?

- [Objective-C implements reference counting in two ways](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html). Also there is garbage collection, but it is deprecated.
- [Vala](https://live.gnome.org/Vala) and gobject (the underlying technology), use [reference counting](https://live.gnome.org/Vala/ReferenceHandling), and also have the concept of a "weak" reference.
- Ruby implements RAII using blocks. This is used across the stdlib: File.open, Dir.mktmpdir, etc. Once you escape the block the resource is closed, deleted, cleaned, etc, depending on the resource type.
- You can simulate some RAII in Java using "finally" in the function body, but does not work for objects passed around.
