## Static (Fixed Size) Queue:

There is almost always a need to have some kind of a queuing functionality. A circular buffer is a good compromise between speed of execution and memory consumption (vs [std::deque](http://en.cppreference.com/w/cpp/container/deque) for example). If your product allows usage of dynamic memory allocation and/or exceptions than [boost::circular_buffer](http://www.boost.org/doc/libs/1_55_0/doc/html/circular_buffer.html) can be a good choice. However, if using dynamic memory allocation is not an option, then there is no other choice but to implement a circular buffer with maximum length known at compile time over C array or [std::array](http://en.cppreference.com/w/cpp/container/array). [Here](https://github.com/arobenko/embxx/blob/master/embxx/container/StaticQueue.h) is the implementation of `StaticQueue` functionality from [embxx](https://github.com/arobenko/embxx) library. I won't go into too much details or explain every line of code. Instead I will emphasise several important points that must be taken into consideration.

### Invalid operations

There can always be an attempt to perform an invalid operation, such as access an element outside the queue boundaries, or inserting new element when the queue is full, or popping an element when queue is empty, etc... The conventional way in C++ to handle these cases is to throw an exception. However, in embedded and especially in bare metal programming it's not an option. The right way to handle these errors would be asserting on pre-conditions. The `StaticQueue` implementation in [embxx](https://github.com/arobenko/embxx) library uses `GASSERT()` macro described earlier. The checks will be compiled only in non-Release mode (NDEBUG not defined) and in case of the failure it will invoke the project specific code the developer has written to report assertion failure.
```cpp
template <typename T, std::size_t TSize> 
class StaticQueue 
{ 
public: 
    ... 
    void popFront() 
    { 
        GASSERT(!empty()); 
        ... 
    } 
}; 
```

### Construction/Destruction of the elements

When the queue is created it doesn't contain any elements. However it must contain uninitialised space where elements can be created in the future. The space must be of sufficient size and be properly aligned.
```cpp
template <typename T, std::size_t TSize> 
class StaticQueue 
{ 
public: 
    typedef T ValueType; 
    ... 
private: 
    typedef 
        typename std::aligned_storage< 
            sizeof(ValueType), 
            std::alignment_of<ValueType>::value 
        >::type StorageType; 

    typedef std::array<StorageType, TSize> ArrayType; 

    ArrayType array_; 
    ... 
}; 
```

When adding a new element to the queue, the “in-place” construction must be performed:
```cpp
template <typename T, std::size_t TSize> 
class StaticQueue 
{ 
public: 
    ... 
    typedef T ValueType; 
    ... 

    template <typename U> 
    void pushBack(U&& newElem) 
    { 
        auto* spacePtr = ...; // get pointer to the right place 
        new (spacePtr) ValueType(std::forward<U>(newElem)); 
        ...
    } 
}; 
```

When an element removed from the queue, explicit destruction must be performed:
```cpp
template <typename T, std::size_t TSize> 
class StaticQueue 
{ 
public: 
    ... 
    typedef T ValueType; 
    ... 
   void popBack() 
    { 
        auto* spacePtr = ...; // get pointer to the right place 
        auto* elemPtr = reinterpret_cast<ValueType*>(spacePtr); 
        elemPtr->~T(); // call the destructor; 
        ... 
    } 
}; 
```

### Iteration
There is often a need to iterate over the elements of the queue. The standard sequential random access containers such as [std::array](http://en.cppreference.com/w/cpp/container/array), [std::vector](http://en.cppreference.com/w/cpp/container/vector) or [std::deque](http://en.cppreference.com/w/cpp/container/deque) may use a simple pointer (or a wrapper class around it) as iterator because  address of every element is greater than address of its predecessor. Incrementing a pointer during the iteration would be enough to get an access to the next element. However, in circular queue/buffer there may be a case when address of the beginning of the queue is greater than address of the end of the queue:

![Non linearised queue image](../image/queue_non_linearised.png)

In this case having a simple pointer as iterator is not enough. There is a need to check a wrap-around case when incrementing an iterator. However always using this kind of iterator may incur undesired performance penalties. That is when “leniarisation” concept pops up. When the queue is linearised, address of every element is greater than the address of its predecessor and simple pointer (linearised iterator) may be used to iterate over all the elements in the queue:

![Linearised queue image](../image/queue_linearised.png)

When the queue is not linearised, it either must be linearised (may be a bit expensive, depending on the size of the queue) or iterate over all the elements in two stages: first on the first (top) part, then on the second (bottom) part. The `StaticQueue` implementation in [embxx](https://github.com/arobenko/embxx) library provides two functions `arrayOne()` and `arrayTwo()` that return these two ranges.

However, there may be a need to read/write data from/to the queue without worrying about the wrap-around case. Good example of such case would be having such circular queue/buffer to contain data read from some communication interface, such as serial port, and there is a need to deserialise 4 byte value from this buffer. The most convenient way would be to use `embxx::io::readBig<4>(iter)` described previously. To properly support this case we will need to have a bit more expensive iterator that properly handles wrap-around when incremented and/or dereferenced. This is the reason for having two types of iterators for `StaticQueue`: `LinearisedIterator` and `Iterator`. The former is a simple `typedef` for a pointer which can be used only on the linearised part of the queue and the latter may be used when iterating without any knowledge whether there is a wrap-around case during the iteration.

When defining a new custom iterator class, there is a need to properly support [std::iterator_traits](http://en.cppreference.com/w/cpp/iterator/iterator_traits) for it. The traits are used to implement functions such as [std::advance](http://en.cppreference.com/w/cpp/iterator/advance) or [std::distance](http://en.cppreference.com/w/cpp/iterator/distance). The requirement is to define the following internal types:
```cpp
template <typename T, std::size_t TSize> 
class StaticQueue 
{ 
public: 
    class Iterator 
    { 
    public: 
        typedef std::random_access_iterator_tag iterator_category; 
        typedef T value_type; 
        typedef T* pointer; 
        typedef T& reference; 
        typedef typename std::iterator_traits<pointer>::difference_type difference_type; 
        ... 
    }; 

    ... 
}; 
```

### Copying queues

Care must be taken when copying/moving elements between the queues. The compiler is not aware of the right type of the elements that are stored in the queue as well as number of valid elements in the queue is unknown at compile time. When using default copy/move constructor and/or assignment operator the compiler will generate a code that copies raw bytes in the storage space between the queues. It may work for the basic type or POD structs, but it is not the right way to do the copying. There is a need to use copy/move constructors in case of constructions or copy/move assignment operator in case of assignment of the valid elements and not copy/move garbage data from unused space.

In addition to regular copy/move constructors and assignment operators, there may also be a need to provide copy/move construction and/or copy/move assignment from the queue that contains elements of the same type, but has different capacity:
```cpp
template <typename T, std::size_t TSize> 
class StaticQueue 
{ 
public: 
    ... 

    template <std::size_t TAnySize> 
    StaticQueue(const StaticQueue<T, TAnySize>& queue) 
        : Base(&array_[0], TSize) 
    { 
        ... // Copy all the elements from other queue 
    } 

    template <std::size_t TAnySize> 
    StaticQueue(StaticQueue<T, TAnySize>&& queue) 
        : Base(&array_[0], TSize) 
    { 
        ... // Move all the elements from other queue 
    } 

    template <std::size_t TAnySize> 
    StaticQueue& operator=(const StaticQueue<T, TAnySize>& queue) 
    { 
        ... // Copy all the elements from other queueu 
    } 

    template <std::size_t TAnySize> 
    StaticQueue& operator=(StaticQueue<T, TAnySize>&& queue) 
    { 
        ... // Move all the elements from other queue 
    } 
    ... 
};
```

### Optimising code generation

As we all know and confirmed in [Templates](../compiler_output/templates.md) chapter, any difference in the value of template parameter will create new instantiation of executable code. It means that having multiple queues of the same type, but different sizes may bloat the executable in an unacceptable way. The best way to solve this problem would be defining a base class that is templated only on the type of the stored values and implements the whole logic of the queue while the derived `StaticQueue` class will just provide the necessary storage area and reuse (wrap) all the functions implemented in the base class:
```cpp
namespace details
{

template <typename T> 
class StaticQueueBase 
{ 
protected: 
    typedef T ValueType; 
    typedef 
        typename std::aligned_storage< 
            sizeof(ValueType), 
            std::alignment_of<ValueType>::value 
        >::type StorageType; 
    typedef StorageType* StorageTypePtr; 

    StaticQueueBase(StorageTypePtr data, std::size_t capacity) 
        : data_(data), 
          capacity_(capacity), 
          startIdx_(0), 
          count_(0) 
    { 
    } 

    template <typename U> 
    void pushBack(U&& value) {...} 

    ... // All other API functions 

private: 
    StorageTypePtr data_; // Pointer to storage area 
    std::size_t capacity_; // Capacity of the storage area 
    std::size_t startIdx_; // Index of the beginning of the queue 
    std::size_t count_; // Number of elements in the queue 
};

} // namespace details

template <typename T, std::size_t TSize> 
class StaticQueue : public details::StaticQueueBase<T> 
{ 
    typedef details::StaticQueueBaseOptimised<T> Base; 
    typedef typename Base::StorageType StorageType; 

public: 
   StaticQueue() 
        : Base(&array_[0], TSize) 
    { 
    } 

    template <typename U> 
    void pushBack(U&& value) 
    { 
        Base::pushBack(std::forward<U>(value)); 
    } 
 
    ... // Wrap all other API functions 
    
private: 
    typedef std::array<StorageType, TSize> ArrayType; 
    ArrayType array_; 
};
```

There are ways to optimise even more. Let's take queues of `int` and `unsigned` values for example. They have the same size and from the queue implementation perspective there is no difference in handling them, so it would be a waste of code space to allow the instantiation of the same binary code for the queue to handle both of these types. Using template specialisation tricks we may implement queues of signed integral types to be a mere wrappers around queues that contain unsigned integral types. Additional example would be storage of the pointers to any types. It would be wise to specialise `StaticQueue` of pointers to be a wrapper around queue of `void*` pointers or even integral unsigned values of the same size as pointers (such as `std::uint32_t` on 32 bit architecture or `std::uint64_t` on 64 bit architecture).

Thanks to the template specialisation there are virtually no limits to optimisations we may apply. However I would like to remind you the well known saying “Premature optimisations are the root of all evil”. Please avoid optimising your `StaticQueue` implementation until the need arises.

