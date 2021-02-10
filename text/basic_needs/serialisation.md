## Data Serialisation

Another essential need in embedded development is an ability to serialise data. Most embedded products read data from some kind of sensors and/or communicate with the control centre via some wired or wireless serial interface. 

Before data is sent via a communication link, it must be serialised into a buffer, and when received, deserialised from bytes also in a different buffer on the other end. The data may be serialised using big or little endian, based on the communication protocol used. The [embxx](https://github.com/arobenko/embxx) library provides a generic code with an ability to read and write integral values from/to any buffer. [Here](https://github.com/arobenko/embxx/blob/master/embxx/io/access.h) is the source code for the functions described below.

The functions below (defined in namespace `embxx::io`) support read and write of an integral value using any type of iterator:
```cpp
template <typename T, typename TIter> 
void writeBig(T value, TIter& iter); 

template <typename T, typename TIter> 
T readBig(TIter& iter); 

template <typename T, typename TIter> 
void writeLittle(T value, TIter& iter); 

template <typename T, typename TIter> 
T readLittle(TIter& iter); 
```

These functions receive reference to iterator of a buffer/container. When bytes are read/written from/to the buffer, the iterator is incremented. The iterator can be of any type as long as it supports dereferencing (`operator*()`), pre-increment (`operator++`) and assignment to dereferenced object. For example, serialising several values of various lengths into the array using big endian:
```cpp
std::uint8_t buf[128];
auto iter = &buf[0];

std::uint16_t value1 = 0x0102;
std::uint32_t value2 = 0x03040506;
std::uint64_t value3 = 0x0708090a0b0c0d0e;

embxx::io::writeBig(value1, iter);
embxx::io::writeBig(value2, iter);
embxx::io::writeBig(value3, iter);
```

The contents of the buffer will be:
`{0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x0a, 0x0b, 0x0c 0x0d, 0x0e, …}`

Similar code of reading values from the buffer would be:
```cpp
std::uint8_t buf[128];
auto iter = &buf[0];

auto value1 = embxx::io::readBig<std::uint16_t>(iter);
auto value2 = embxx::io::readBig<std::uint32_t>(iter);
auto value3 = embxx::io::readBig<std::uint64_t>(iter);
```

Another example is serialising data into a container that has `push_back()` member functions, such as [std::vector](http://en.cppreference.com/w/cpp/container/vector) or circular buffer. The data will be added at the end of the existing one:
```cpp
std::vector<std::uint8_t> buf;
auto iter = std::back_inserter(buf); // Will call push_back 
                                     // on assignment
…
// The writes below will use push_back for every byte.
embxx::io::writeBig(value1, iter); 
embxx::io::writeBig(value2, iter);
embxx::io::writeBig(value3, iter);
```

Depending on a communication protocol there may be a need to serialise only part of the value. For example some field of communication protocol is defined having only 3 bytes. In this case the value will probably be stored in a variable of `std::uint32_t` type. There is similar set of functions, but with additional template parameter that specifies how many bytes to read/write:
```cpp
template <std::size_t TSize, typename T, typename TIter> 
void writeBig(T value, TIter& iter); 

template <typename T, std::size_t TSize, typename TIter> 
T readBig(TIter& iter); 

template <std::size_t TSize, typename T, typename TIter> 
void writeLittle(T value, TIter& iter); 

template <typename T, std::size_t TSize, typename TIter> 
T readLittle(TIter& iter); 
```

So to read/write 3 bytes will look like the following:
```cpp
auto value = embxx::io::readBig<std::uint32_t, 3>(iter);
embxx::io::writeBig<3>(value, iter);
```

Sometimes the endianness of data serialisation may depend on some traits class parameters. In order to be able to choose “Little” or “Big” variant functions at compile time instead of runtime the tag parameter dispatch idiom must be used.

There are similar read/write functions, but instead of being differentiated by name they have additional tag parameter to specify the endianness of serialisation:
```cpp
/// Same as writeBig<T, TIter>(value, iter); 
template <typename T, typename TIter> 
void writeData( 
    T value, 
    TIter& iter, 
    const traits::endian::Big& endian); 

/// Same as writeBig<TSize, T, TIter>(value, iter) 
template <std::size_t TSize, typename T, typename TIter> 
void writeData( 
    T value, 
    TIter& iter, 
    const traits::endian::Big& endian); 

/// Same as writeLittle<T, TIter>(value, iter) 
template <typename T, typename TIter> 
void writeData( 
    T value, 
    TIter& iter, 
    const traits::endian::Little& endian); 

/// Same as writeLittle<TSize, T, TIter>(value, iter) 
template <std::size_t TSize, typename T, typename TIter> 
void writeData( 
    T value, 
    TIter& iter, 
    const traits::endian::Little& endian); 

/// Same as readBig<T, TIter>(iter) 
template <typename T, typename TIter> 
T readData(TIter& iter, const traits::endian::Big& endian); 
 
/// Same as readBig<TSize, T, TIter>(iter) 
template <typename T, std::size_t TSize, typename TIter> 
T readData(TIter& iter, const traits::endian::Big& endian); 

/// Same as readLittle<T, TIter>(iter) 
template <typename T, typename TIter> 
T readData(TIter& iter, const traits::endian::Little& endian); 

/// Same as readLittle<TSize, T, TIter>(iter) 
template <typename T, std::size_t TSize, typename TIter> 
T readData(TIter& iter, const traits::endian::Little& endian); 
```

The `traits::endian::Big` and `traits::endian::Little` are defined as empty tag classes:
```cpp
namespace traits 
{ 

namespace endian 
{ 

struct Big {}; 

struct Little {}; 

}  // namespace endian 

}  // namespace traits 
```

For example:
```cpp
template <typename TTraits>
class SomeClass
{
public:
    typedef typename TTraits::Endianness Endianness;

    template <typename TIter>
    void serialise(TIter& iter) const
    {
        embxx::io::writeData(data_, iter, Endianness());
    }

private:
    std::uint32_t data_;
};
```

So the code above is not aware what endianness is used to serialise the data. It is provided as internal type of `Traits` class named `Endianness`. The compiler will generate the call to appropriate `writeData()` function, which in turn forward it to `writeBig()` or `writeLittle()`.

To serialise data using big endian the traits should be defined as following:
```cpp
struct MyTraits
{
    typedef embxx::io::traits::endian::Big Endianness;
};

SomeClass<MyTraits> someClassObj;
…
someClassObj.serialise(iter); // Will serialise using big endian
```

The interface described above is very easy and convenient to use and quite easy to implement using straightforward approach. However, any variation of template parameters create an instantiation of new binary code which may create significant code bloat if not used carefully. Consider the following:
* Read/write of signed vs unsigned integer values. The serialisation/deserialisation code is identical for both cases, but won't be considered as such when instantiating the functions. To optimise this case, there is a need to implement read/write operations only for unsigned value, while the “signed” functions become wrappers around the former. Don't forget a sign extension operation when retrieving partial signed value.
* The read/write operations are more or less the same for any length of the values, i.e of any types: `(unsigned) char`, `(unsigned) short`, `(unsigned) int`, etc... To optimise this case, there is a need for internal function that receives length of serialised value as a run time parameter, while the functions described above are mere wrappers around it.
* Usage of the iterators also require caution. For example reading values may be performed using regular `iterator` as well as `const_iterator`, i.e. iterator pointing to const values. These are two different iterator types that will duplicate the “read” functionality if both of them are used:

```cpp
char buf[128] = {…};
const char* iter1 = &buf[0];
char* iter2 = &buf[0];

// Instantiation 1
auto value1 = embxx::io::readBig<std::uint16_t>(iter1); 

// Instantiation 2
auto value2 = embxx::io::readBig<std::uint16_t>(iter2); 
```
It is possible to optimise the case above for random access iterator by using temporary pointers to unsigned characters to read the required value. After retrieval is complete, just increment the value of the passed iterator with number of characters read.

All the consideration points stated above require quite complex implementation of the serialisation/deserialisation functionality with multiple levels of abstraction which is beyond the scope of this book. It would be a nice exercise to try and implement it yourself. Another option is to use the code as is from [embxx](https://github.com/arobenko/embxx) library.

