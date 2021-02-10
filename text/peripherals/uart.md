## UART

Our next stage will be to support debug logging via UART interface. In conventional C++ logging is performed using either [printf](http://en.cppreference.com/w/cpp/io/c/fprintf) function or [output streams](http://en.cppreference.com/w/cpp/io/basic_ostream) (such as [std::cout](http://en.cppreference.com/w/cpp/io/cout) or [std::cerr](http://en.cppreference.com/w/cpp/io/cerr)).

If `printf` is used the compilation may fail at the linking stage with following errors:
```
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-sbrkr.o): In function `_sbrk_r':
sbrkr.c:(.text._sbrk_r+0x18): undefined reference to `_sbrk'
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-writer.o): In function `_write_r':
writer.c:(.text._write_r+0x20): undefined reference to `_write'
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-closer.o): In function `_close_r':
closer.c:(.text._close_r+0x18): undefined reference to `_close'
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-fstatr.o): In function `_fstat_r':
fstatr.c:(.text._fstat_r+0x1c): undefined reference to `_fstat'
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-isattyr.o): In function `_isatty_r':
isattyr.c:(.text._isatty_r+0x18): undefined reference to `_isatty'
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-lseekr.o): In function `_lseek_r':
lseekr.c:(.text._lseek_r+0x20): undefined reference to `_lseek'
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-readr.o): In function `_read_r':
readr.c:(.text._read_r+0x20): undefined reference to `_read'
collect2: error: ld returned 1 exit status
```

Once these functions are stubbed with empty bodies, the compilation will succeed, but the image size will be quite big (around 45KB). 

The `_sbrk` function is required to support dynamic memory allocation. The `printf` function probably uses `malloc()` to allocate some temporary buffers. If we open the assembly listing file we will see calls to `<malloc>` and `<free>`. 

The `_write` function is used to write characters into the standard output consol, which doesn't exist in embedded product. The developer must use this function implementation to write all the provided characters to UART serial interface. Many developers implement this function in a straightforward synchronous way with busy loop:
```cpp
extern "C" int _write(int file, char *ptr, int len)
{
    int count = len;
    if (file == 1) { // stdout
        while (count > 0) {
            while (... /* poll the status bit */) {} // just wait
            TX_REG = *ptr;
            ++ptr;
            --count;
        }
    }
    return len;
}
```

In this case the call to `printf` function will be blocking and won't return until all the characters are written one by one to UART, which takes a lot of execution time. This approach is suitable for quick and dirty debugging, but will quickly become impractical when the project grows.

In order to make the execution of `printf` quick, there must be some kind of interrupt driven component that is responsible to buffer all the provided characters and forward it to UART asynchronously one by one using "**TX buffer register is free**" kind of interrupts.

One of disadvantages in using `printf` for logging is a necessity to specify an output format of the printed variables:
```cpp
std::int32_t i = ...; // some value
printf("Value = %d\n");
```

In case the type of the printed variable changes, the developer must remember to update type in the format string too. This is the reason why many C++ developers prefer using streams instead of `printf`:
```cpp
std::int32_t i = ...; // some value
std::cout << "Value = " << i << std::endl;
```
Even if type of printed variable changes the compiler will generate a call to appropriate overloaded `operator<<` of [std::ostream](http://en.cppreference.com/w/cpp/io/basic_ostream) and the value will be printed correctly. The developer will also have to implement the missing `_write` function to write provided characters somewhere (UART interface in our case).

However using C++ streams in bare metal development is often not an option. They use exceptions to handle error cases as well as [locales](http://en.cppreference.com/w/cpp/locale/locale) for formatting. The compilation of simple output statement with streams above created image of more than 500KB using [GNU Tools for ARM Embedded Processors](https://launchpad.net/gcc-arm-embedded) compiler. 

To summarise all the stated above, there may be a problem to use standard [printf](http://en.cppreference.com/w/cpp/io/c/fprintf) function or [output streams](http://en.cppreference.com/w/cpp/io/basic_ostream) for debug logging, especially in systems with small memory and where dynamic memory allocations and exceptions mustn't be used. Our ultimate goal will be creation of standard output stream like interface for debug logging while using asynchronous event handling with [Device-Driver-Component](../basic_concepts/device_driver_component.md) model and [Event Loop](../basic_concepts/event_loop.md) where most of the code is generic and only smal part of managing write of a single character to the UART interface is platform specific.

Asyncrhonous read and write operations on the UART interface are very similar to the generic way of programming and handling asynchronous events described earlier in [Device-Driver-Component](../basic_concepts/device_driver_component.md) chapter.

### Writing to UART

**Stage1** - Sending asynchronous buffer write request from the **Component** layer to **Driver** in event loop (non-interrupt) context.

![Image: Asyncrhonous write request](../image/character_driver_flow_write1.png)

The **Component** calls `asyncWrite()` member function of the **Driver** and provides pointer to the buffer, size of the buffer and the callback object to invoke when the write is complete. The `asyncWrite()` function needs to be able to receive any type of callable object, such as [std::bind](http://en.cppreference.com/w/cpp/utility/functional/bind) expression or [lambda function](http://en.cppreference.com/w/cpp/language/lambda). To achieve this the function must be templated:
```cpp
class CharacterDriver
{
public:
    typedef ... CharType;

    template <typename TCallbackFunc>
    void asyncWrite(
        const CharType* buf, 
        std::size_t bufSize, 
        TCallbackFunc&& func);
};
```
According to the convention mentioned [earlier](../basic_concepts/device_driver_component.md), the callback must receive an error status of whether the operation is successful as its first parameter. When performing asynchronous operation on the buffer, it can be required to know how many characters have been read / written before the error occurred, in case the operation wasn't successful. For this purpose such callback object must receive number of bytes written as the second parameter, i.e. expose the `void (const embxx::error::ErrorStatus& err, std::size_t bytesTransferred)` signature.

When the **Driver** receives the asynchronous operation request, it forwards it to the **Device**, letting the latter know how many bytes will be written during the whole process. Please note that **Driver** uses [embxx::device::context::EventLoop](https://dl.dropboxusercontent.com/u/46999418/embxx/structembxx_1_1device_1_1context_1_1EventLoop.html) tag parameter to specify that `startWrite()` member function of **Device** is invoked in event loop (non-interrut) context. The job of the **Device** object is to enable appropriate interrupts and return immediately. Once the interrupt occurs, the stage of writing the data begins.

**Stage2** - Writing provided data.

![Image: Writing provided data](../image/character_driver_flow_write2.png)

Once the interrupt of "TX available" occurs, the **Device** must let the **Driver** know. There must obviously be some kind of callback involved, which **Driver** must provide during its construction / initialisation stage. Let's assume at this moment that such assignment was successfully done, and **Device** is capable of successfully notifying the **Driver**, that there is an ability to write character to TX FIFO of the peripheral.

When the **Driver** receives such notification, it attempts to write as many characters as possible:
```cpp
typedef embxx::device::context::Interrupt InterruptContext;

void canWriteCallback()
{
    // Executed in interrupt context, must be quick
    while(device_.canWrite(InterruptContext())) {
        if ((writeBufStart_ + writeBufSize_) <= currentWriteBufPtr_) {
            break;
        }

        device_.write(*currentWriteBufPtr_, InterruptContext());
        ++currentWriteBufPtr_;
    }
}
```
This is because when "TX available" interrupt occurs, there may be a place for multiple characters to be sent, not just one. Doing checks and writes in a loop may save many CPU cycles.

Please note, that all these calls are performed in interrupt context. They are marked in red in the picture above.

Once the Tx FIFO of the underlying **Device** is full or there are no more characters to write, the callback returns. The whole cycle described above is repeated on every "TX available" interrupt until the whole provided buffer is sent to the **Device** for writing.

**Stage3** - Notifying caller about completion:

Once the whole buffer is sent to the **Device** for writing, the **Driver** is aware that there will be no more writes performed. However it doesn't report completion until the **Device** itself calls appropriate callback indicating that the operation has been indeed completed. Shifting the responsibility of identifying when the operation is complete to **Device** will be needed later when we will want to reuse the same **Driver** for [I2C](i2c.md) and [SPI](spi.md) peripherals. It will be important to know when internal Tx FIFO of the peripheral becomes empty after all the characters from previous operation have been written.

![Image: Notifying caller about completion](../image/character_driver_flow_write3.png)

Once the **Driver** receives notification from the **Device** (still in interrupt context), that the write operation is complete, it bundles the callback object, provided with initial `asyncWrite()` request, together with error status and number of actual bytes transferred using [std::bind](http://en.cppreference.com/w/cpp/utility/functional/bind) expression and sends the callable object to [Event Loop](../basic_concepts/event_loop.md) for execution in event loop (non-interrupt) context. 

### Reading from UART

The reading from UART is done in a very similar manner.

**Stage1** - Sending asynchronous buffer read request from the **Component** layer to **Driver** in event loop (non-interrupt) context.

![Image: Asynchronous read request](../image/character_driver_flow_read1.png)

The `asyncRead()` member function of the **Driver** should allow callback to be callable object of any type (but one that exposes predefined signature of course).

```cpp
class CharacterDriver
{
public:
    typedef ... CharType;

    template <typename TCallbackFunc>
    void asyncRead(
        CharType* buf, 
        std::size_t bufSize, 
        TCallbackFunc&& func);
};
```

**Stage2** - Reading data into the buffer.

![Image: Writing provided data](../image/character_driver_flow_read2.png)

The callback's implementation will be something like:
```cpp
    void canReadCallback()
    {
        while(device_.canRead(InterruptContext())) {
            if ((readBufStart_ + readBufSize_) <= currentReadBufPtr_) {
                break;
            }

            auto ch = device_.read(InterruptContext());
            *currentReadBufPtr_ = ch;
            ++currentReadBufPtr_;
        }
    }
```

**Stage3** - Notifying caller about completion:

![Image: Notifying caller about completion](../image/character_driver_flow_read3.png)

### Cancelling Asynchronous Operations

The cancellation flow is very similar to the one described in [Device-Driver-Component](../basic_concepts/device_driver_component.md) chapter:

![Image: Cancel read](../image/character_driver_cancel_read1.png)

If the cancellation is successful, the callback must be invoked with error code indicating that the operation was aborted (`embxx::error::ErrorCode::Aborted`).

One possible case of unsuccessful cancellation is when callback was posted for execution in event loop, but hasn't been executed yet when cancellation is attempted. In this case **Driver** is aware that there is no pending asynchronous operation and can return `false` immediately.

![Image: Cancel read](../image/character_driver_cancel_read2.png)

Another possible case of unsuccessful cancellation is when completion interrupt occurs in the middle of cancellation request:

![Image: Cancel read](../image/character_driver_cancel_read3.png)

### Reading "Until"

There may be a case, when partial read needs to be performed, for example until specific character is encountered. In this case the **Driver** is responsible to monitor incoming characters and cancel the read into the buffer operation before its completion:

![Image: Notifying caller about completion](../image/character_driver_flow_read_until.png)

Note, that previously **Driver** called `cancelRead()` member function of the **Device** in event loop (non-interrupt) context, while in "read until" situation the cancellation happens in interrupt mode. That requires **Device** to implement these functions for both modes:
```cpp
class MyDevice
{
public:
    bool cancelRead(embxx::device::context::EventLoop) {...}
    bool cancelRead(embxx::device::context::Interrupt) {...}
};
```

The `asyncReadUntil()` member function of the **Driver** should be able to receive any stateless predicate object that defines `bool operator()(CharType ch) const`. The predicate invocation should return true when expected character is received and reading operation must be stopped.
```cpp
class MyDriver
{
public:
    template <typename TPred, typename TFunc>
    void asyncReadUntil(
        CharType* buf,
        std::size_t size,
        TPred&& pred,
        TFunc&& func)
    { 
        ...
    }
};
```
It allows using complex conditions in evaluating the character. For example, stopping when either '\r' or '\n' is encountered:
```cpp
typedef embxx::error::ErrorStatus EmbxxErrorStatus;

driver_.asyncReadUntil(
    buf, 
    bufSize, 
    [](CharType ch) -> bool 
        {
            return (ch == '\r') || (ch == '\n');
        }, 
    [](const EmbxxErrorStatus& es, std::size_t bytesTransferred)
        {
            ...
        });
```

### Device Implementation

In this section I will try to describe in more details what **Device** class needs to provide for the **Driver** to work correctly. First of all it needs to define the type of characters used:
```cpp
class MyDevice
{
public:
    typedef std::uint8_t CharType;
};
```

The **Driver** layer will reuse the definition of the character in its internal functions:
```cpp

template<typename TDevice, ...>
class MyDriver
{
public:
    typedef typename TDevice::CharType CharType;

    void asyncRead(CharType* buf, std::size_t bufSize, ...) {}
};
```

There is a need for **Device** to be able to record callback objects from the **Driver** in order to notify the latter about an ability to read/write next character and about operation completion. 
```cpp
class MyDevice
{
public:
    template <typename TFunc>
    void setCanReadHandler(TFunc&& func)
    {
        canReadHandler_ = std::forward<TFunc>(func);
    }

    template <typename TFunc>
    void setCanWriteHandler(TFunc&& func)
    {
        canWriteHandler_ = std::forward<TFunc>(func);
    }

    template <typename TFunc>
    void setReadCompleteHandler(TFunc&& func)
    {
        readCompleteHandler_ = std::forward<TFunc>(func);
    }

    template <typename TFunc>
    void setWriteCompleteHandler(TFunc&& func)
    {
        writeCompleteHandler_ = std::forward<TFunc>(func);
    }

private:
    typedef ... OpAvailableHandler;
    typedef ... OpCompleteHandler;

    OpAvailableHandler canReadHandler_;
    OpCompleteHandler readCompleteHandler_;

    OpAvailableHandler canWriteHandler_;
    OpCompleteHandler writeCompleteHandler_;

};
```

The `OpAvailableHandler` and `OpCompleteHandler` type may be either hard coded to be `std::function<void ()>` and `std::function<void (const embxx::error::ErrorStatus&)>` respectively or passed as template parameters:
```cpp
template <typename TCanReadHandler,
          typename TCanWriteHandler,
          typename TReadCompleteHandler,
          typename TWriteCompleteHandler>
class MyDevice
{
public:
    ... // setters are as above

private:

    TCanReadHandler canReadHandler_;
    TReadCompleteHandler readCompleteHandler_;

    TCanWriteHandler canWriteHandler_;
    TWriteCompleteHandler writeCompleteHandler_;
};
```

Choosing the "template parameters option" is useful when the same **Device** class is reused between multiple applications for the same product line.

The next stage would be implementing all the required functions:
```cpp
class MyDevice
{
public:
    
    typedef embxx::device::context::EventLoop EventLoopContext;
    typedef embxx::device::context::Interrupt InterruptContext;

    // Start read operation - enables interrupts
    void startRead(std::size_t length, EventLoopContext context);

    // Cancel read in event loop context
    bool cancelRead(EventLoopContext context);

    // Cancel read in interrupt context - used only if 
    // asyncReadUntil() function was used in Device
    bool cancelRead(InterruptContext context);

    // Start write operation - enables interrupts
    void startWrite(std::size_t length, EventLoopContext context);

    // Cancell write operation
    bool cancelWrite(EventLoopContext context);

    // Check whether there is a character available to be read.
    bool canRead(InterruptContext context);

    // Check whether there is space for one character to be written.
    bool canWrite(InterruptContext context);

    // Read the available character from Rx FIFO of the peripheral
    CharType read(InterruptContext context);

    // Write one more character to Tx FIFO of the peripheral
    void write(CharType value, InterruptContext context);
};
```

Note, that there may be extra configuration functions specific for the peripheral being controlled. For example baud rate, parity, flow control for UART. Such configuration is almost always platform and/or product specific and usually performed at application startup. It is irrelevant to the [Device-Driver-Component](../basic_concepts/device_driver_component.md) model introduced in this book.

```cpp
class MyDevice
{
public:
    void configBaud(unsigned value) { ... }
    ...
};
```

The [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project has multiple applications that use UART1 interface for logging. The peripheral control code is the same for all of them and is implemented in [src/device/Uart1.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/Uart1.h).

### Driver Implementation

**Driver** must be a generic piece of code, that can be reused with any **Device** control object (as long as it exposed right public interface) and in any application, including ones without dynamic memory allocation.

First of all, we will need references to **Device** as well as **[Event Loop](../basic_concepts/event_loop.md)** objects:
```cpp
template <typename TDevice, typename TEventLoop>
class MyDriver
{
public:
    // Reuse definition of character type from the Device
    typedef TDevice::CharType CharType;

    // During the construction store references to Device
    // and Event Loop objects.
    MyDriver(TDevice& device, TEventLoop& el)
      : device_(device),
        el_(el)
    {
        // Register appropriate callbacks with device
        device_.setCanReadHandler(
            std::bind(
                &MyDriver::canReadInterruptHandler, this));
        device_.setReadCompleteHandler(
            std::bind(
                &MyDriver::readCompleteInterruptHandler,
                this,
                std::placeholders::_1));

        device_.setCanWriteHandler(
            std::bind(
                &MyDriver::canWriteInterruptHandler, this));
        device_.setWriteCompleteHandler(
            std::bind(
                &MyDriver::writeCompleteInterruptHandler,
                this,
                std::placeholders::_1));

    }

    ...

private:

    void canReadInterruptHandler() {...}
    void readCompleteInterruptHandler(
        const embxx::error::ErrorStatus& es) {...}

    void canWriteInterruptHandler() {...}
    void writeCompleteInterruptHandler(
        const embxx::error::ErrorStatus& es) {...}

    TDevice& device_;
    TEventLoop& el_;
};
```

We will also need to store callbacks provided with any asynchronous operation. Note that the "read" and "write" are independent operations and it should be possible to perform `asyncRead()` and `asyncWrite()` calls at the same time. 

The only way to make **Driver** generic is to move responsibility of specifying callback storage type up one level, i.e. we must put them as template parameters:
```cpp
template <typename TDevice, 
          typename TEventLoop,
          typename TReadCompleteCallback,
          typename TWriteCompleteCallback>
class MyDriver
{
public:
    ...

    typedef embxx::device::context::EventLoop EventLoopContext;

    template <typename TFunc>
    void asyncRead(
        CharType* buf,
        std::size_t bufSize,
        TFunc&& func)
    {
        readBufStart_ = buf;
        currentReadBufPtr = buf;
        readBufSize_ = bufSize;
        readCompleteCallback_ = std::forward<TFunc>(func);
        driver_.startRead(bufSize, EventLoopContext());
    }

    template <typename TFunc>
    void asyncWrite(
        const CharType* buf,
        std::size_t bufSize,
        TFunc&& func)
    {
        writeBufStart_ = buf;
        currentWriteBufPtr = buf;
        writeBufSize_ = bufSize;
        writeCompleteCallback_ = std::forward<TFunc>(func);
        driver_.startWrite(bufSize, EventLoopContext());
    }

private:
    ...

    // Read info
    CharType* readBufStart_;
    CharType* currentReadBufPtr_;
    std::size_t readBufSize_;
    TReadCompleteCallback readCompleteCallback_;

    // Write info
    const CharType* writeBufStart_;
    const CharType* currentWriteBufPtr_;
    std::size_t writeBufSize_;
    TWriteCompleteCallback writeCompleteCallback_;
};
```

As it was mentioned earlier in [Reading "Until"](#reading-until-) section, there is quite often a need to stop reading characters into the provided buffer when some condition evaluates to true. It means there is also a need to provide storage for the character evaluation predicate:

```cpp
template <typename TDevice, 
          typename TEventLoop,
          typename TReadCompleteCallback,
          typename TWriteCompleteCallback,
          typename TReadUntilPred>
class MyDriver
{
public:
    ...

    typedef embxx::device::context::EventLoop EventLoopContext;

    template <typename TPred, typename TFunc>
    void asyncReadUntil(
        CharType* buf,
        std::size_t bufSize,
        TPred&& pred,
        TFunc&& func)
    {
        readBufStart_ = buf;
        currentReadBufPtr = buf;
        readBufSize_ = bufSize;
        readCompleteCallback_ = std::forward<TFunc>(func);
        readUntilPred_ = std::forward<TPred>(pred)
        driver_.startRead(bufSize, EventLoopContext());
    }

private:
    ...

    // Read info
    CharType* readBufStart_;
    CharType* currentReadBufPtr_;
    std::size_t readBufSize_;
    TReadCompleteCallback readCompleteCallback_;
    TReadUntilPred readUntilPred_;

    ...
};
```

The example code above may work, but it contradicts to one of the basic principles of C++: "You should pay only for what you use". In case of using UART for logging, there is no input from the peripheral and it is a waist to keep data members for "read" required to manage "read" operations. Let's try to improve the situation a little bit by using template specialisation as well as reduce number of template parameters by using "Traits" aggregation struct.
```cpp
struct MyOutputTraits
{
    // The "read" handler storage type.
    typedef std::nullptr_t ReadHandler;

    // The "write" handler storage type.
    // The valid handler must have the following signature:
    //  "void handler(const embxx::error::ErrorStatus&, std::size_t);"
    typedef embxx::util::StaticFunction<
        void(const embxx::error::ErrorStatus&, std::size_t)> WriteHandler;

    // The "read until" predicate storage type
    typedef std::nullptr_t ReadUntilPred;

    // Read queue size
    static const std::size_t ReadQueueSize = 0;

    // Write queue size
    static const std::size_t WriteQueueSize = 1;
};
```

Please note, that allowed number of pending "read" requests is specified as 0 in the traits struct above, i.e. the read operations are not allowed. The "read complete" and "read until predicate" types are irrelevant and specified as [std::nullptr_t](http://en.cppreference.com/w/cpp/types/nullptr_t). The instantiation of the **Driver** object must take it into account and not include any "read" related functionality. In order to achieve this the **Driver** class needs to have two independent sub-functionalities of "read" and "write". It may be achieved by inheriting from two base classes. 
```cpp
template <typename TDevice,
          typename TEventLoop,
          typename TTraits = MyOutputTraits>
class MyDriver :
    public ReadSupportBase<
                TDevice, 
                TEventLoop, 
                typename TTraits::ReadHandler, 
                typename TTraits::ReadUntilPred, 
                TTraits::ReadQueueSize>,
    public WriteSupportBase<
                TDevice, 
                TEventLoop, 
                typename TTraits::WriteHandler, 
                TTraits::WriteQueueSize>
{
    typedef ReadSupportBase<...> ReadBase;
    typedef WriteSupportBase<...> WriteBase;
public:
    template <typename TPred, typename TFunc>
    void asyncRead(
        CharType* buf,
        std::size_t bufSize,
        TFunc&& func)
    {
        ReadBase::asyncRead(buf, bufSize, std::forward<TFunc>(func);
    }

    template <typename TPred, typename TFunc>
    void asyncWrite(
        const CharType* buf,
        std::size_t bufSize,
        TFunc&& func)
    {
        WriteBase::asyncWrite(buf, bufSize, std::forward<TFunc>(func);
    }
};
```

Now, the template specialisation based on queue size should do the job:
```cpp
template <typename TDevice,
          typename TEventLoop,
          typename TReadHandler,
          typename TReadUntilPred,
          std::size_t ReadQueueSize>;
class ReadSupportBase;


template <typename TDevice,
          typename TEventLoop,
          typename TReadHandler,
          typename TReadUntilPred>;
class ReadSupportBase<TDevice, TEventLoop, TReadHandler, TReadUntilPred, 1>
{
public:
    ReadSupportBase(TDevice& device, TEventLoop& el) {...}
    ... // Implements the "read" related API
private:
    ... // Read related data members
};

template <typename TDevice,
          typename TEventLoop,
          typename TReadHandler,
          typename TReadUntilPred>;
class ReadSupportBase<TDevice, TEventLoop, TReadHandler, TReadUntilPred, 0>
{
public:
    ReadSupportBase(TDevice& device, TEventLoop& el) {}
    // No need for any "read" related API and data members
};

template <typename TDevice,
          typename TEventLoop,
          typename TWriteHandler,
          std::size_t WriteQueueSize>;
class WriteSupportBase;


template <typename TDevice,
          typename TEventLoop,
          typename TReadHandler>;
class WriteSupportBase<TDevice, TEventLoop, TWriteHandler, 1>
{
public:
    WriteSupportBase(TDevice& device, TEventLoop& el) {...}
    ... // Implements the "write" related API
private:
    ... // Write related data members
};

template <typename TDevice,
          typename TEventLoop,
          typename TWriteHandler>;
class WriteSupportBase<TDevice, TEventLoop, TWriteHandler, 0>
{
public:
    WriteSupportBase(TDevice& device, TEventLoop& el) {}
    // No need for any "write" related API and data members
};
```

Note, that it is possible to implement general case when read/write queue size is greater than 1. It will require some kind of request queuing (using [Static (Fixed Size) Queue](../basic_needs/queue.md) for example) and will allow issuing multiple asynchronous read/write requests at the same time.

In order to support this extension, the **Device** class must implement some extra functionality too:
1. The new read/write request can be issued by the **Driver** in interrupt context, after previous operation reported completion.
```cpp
class MyDevice
{
public:
    void startRead(std::size_t length, InterruptContext context);
    void startWrite(std::size_t length, InterruptContext context);
};
```
1. When new asynchronous read/write request is issued to the **Driver** it must be able to prevent interrupt context callbacks from being invoked to avoid races on the internal data structure:
```cpp
class MyDevice
{
public:
    bool suspendRead(EventLoopContext context);
    void resumeRead(EventLoopContext context)
    bool suspendWrite(EventLoopContext context);
    void resumeWrite(EventLoopContext context);
};
```
Please pay attention to the boolean return value of `suspend*()` functions. They are like `cancel*()` ones, there is an indication whether the invocation of the callbacks is suspended or there is no operation currently in progress.

Such generic **Driver** is already implemented in [embxx/driver/Character.h](https://github.com/arobenko/embxx/blob/master/embxx/driver/Character.h) file of [embxx](https://github.com/arobenko/embxx) library. The **Driver** is called "Character", because it reads/writes the provided buffer one character at a time. The documentation can be found [here](https://dl.dropboxusercontent.com/u/46999418/embxx/driver_character_page.html).

### Character Echo Application

Now, it is time to do something practical. The [app_uart1_echo](https://github.com/arobenko/embxx_on_rpi/tree/master/src/app/app_uart1_echo) application in [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project implements simple single character echo.

The `System` class in [System.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/app/app_uart1_echo/System.h) file defines the **Device** and **Driver** layers:
```cpp
class System
{
public:
    static const std::size_t EventLoopSpaceSize = 1024;
    typedef embxx::util::EventLoop<
        EventLoopSpaceSize,
        device::InterruptLock,
        device::WaitCond> EventLoop;

    typedef device::InterruptMgr<> InterruptMgr;

    typedef device::Uart1<InterruptMgr> Uart;

    typedef embxx::driver::Character<Uart, EventLoop> UartSocket;

    ...

private:

    ...
    EventLoop el_;
    Uart uart_;
    UartSocket uartSocket_;
};

```
Note that `UartSocket` uses default "TTraits" template parameter of `embxx::driver::Character`, which is defined to be:
```cpp
struct DefaultCharacterTraits
{
    typedef embxx::util::StaticFunction<
        void(const embxx::error::ErrorStatus&, std::size_t)> ReadHandler;
    typedef embxx::util::StaticFunction<
        void(const embxx::error::ErrorStatus&, std::size_t)> WriteHandler;
    typedef std::nullptr_t ReadUntilPred;
    static const std::size_t ReadQueueSize = 1;
    static const std::size_t WriteQueueSize = 1;
};
```
It allows usage of both "read" and "write" operations at the same time. Having the definitions in place it is quite easy to implement the "echo" functionality:
```cpp
// Forward declaration
void writeChar(System::UartSocket& uartSocket, System::Uart::CharType& ch);

void readChar(System::UartSocket& uartSocket, System::Uart::CharType& ch)
{
    uartSocket.asyncRead(&ch, 1,
        [&uartSocket, &ch](const embxx::error::ErrorStatus& es, std::size_t bytesRead)
        {
            GASSERT(!es);
            GASSERT(bytesRead == 1);
            static_cast<void>(es);
            static_cast<void>(bytesRead);
            writeChar(uartSocket, ch);
        });
}

void writeChar(System::UartSocket& uartSocket, System::Uart::CharType& ch)
{
    uartSocket.asyncWrite(&ch, 1,
        [&uartSocket, &ch](const embxx::error::ErrorStatus& es, std::size_t bytesWritten)
        {
            GASSERT(!es);
            GASSERT(bytesWritten == 1);
            static_cast<void>(es);
            static_cast<void>(bytesWritten);
            readChar(uartSocket, ch);
        });
}

int main() {
    auto& system = System::instance();
    auto& uart = system.uart();

    // Configure serial interface
    uart.configBaud(115200);
    uart.setReadEnabled(true);
    uart.setWriteEnabled(true);

    // Start with asynchronous read
    auto& uartSocket = system.uartSocket();
    System::Uart::CharType ch = 0;
    readChar(uartSocket, ch);

    // Run the event loop
    device::interrupt::enable();
    auto& el = system.eventLoop();
    el.run();

    GASSERT(0); // Mustn't exit
    return 0;
```

### Stream-like Printing Interface

As was mentioned earlier, our ultimate goal would be having standard output stream like interface for debug output, which works asynchronously without any blocking busy waits. Such interface must be a generic **Component**, which works in non-interrupt context, while using recently covered generic "Character" **Driver** in conjunction with platform specific "Uart" **Device**.

Such **Component** should be implemented as two sub-**Components**. One is "Stream Buffer" which is responsible to maintain circular buffer of written characters and flush them to the peripheral using "Character" **Driver** when needed. The characters, that have been successfully written, are removed from the internal buffer. The second one is "Stream" itself, which is responsible to convert various values into characters and write them to the end of the "Stream Buffer".

![Image: Stream](../image/stream.png)

Let's start with "Output Stream Buffer" first. It needs to receive reference to the **Driver** it's going to use:
```cpp
template <typename TDriver>
class OutStreamBuf
{
public:
    OutStreamBuf(TDriver& driver)
      : driver_(driver)
    {
    }

private:
   TDriver& driver_;
   ...
};
```

There is also a need to have a buffer, where characters are stored before they are written to the device. Remember that we are trying to create a **Component**, which can be reused in multiple independent projects, including ones that do not support dynamic memory allocation. Hence, [Static (Fixed Size) Queue](../basic_needs/queue.md) may be a good choice for it. It means, there is a need to provide size of the buffer as one of the template arguments:
```cpp
template <typename TDriver,
          std::size_t TBufSize>
class OutStreamBuf
{
public:
   typedef typename TDriver::CharType CharType;
   typedef embxx::container::StaticQueue<CharType, BufSize> Buffer;

private:
   ...
   Buffer buf_;
};
```

The "Output Stream Buffer" needs to support two main operations:
1. Pushing new single character at the end of the buffer.
2. Flushing all (or part of) written characters, i.e. activate asynchronous write with **Driver**.

When pushing a new character, there may be a case when the internal buffer is full. In this case, the pushed character needs to be discarded and there must be an indication whether "push" operation was successful. The function may return either `bool` to indicate success of the operation or `std::size_t` to inform the caller how may characters where written. If `0` is returned, the character wasn't written.
```cpp
template <...>
class OutStreamBuf
{
public:
   // Add new character at the end of the buffer
   std::size_t pushBack(CharType ch);

   // Get number of written, not-flushed characters
   std::size_t size();

   // Flush number of characters
   void flush(std::size_t count = size());
   ...
};
```

This limited number of operations is enough to implement "Output Stream" - like interface. However, "Output Stream Buffer" can be useful in writing any serialised data into the peripheral, not only the debug output. For example using standard algorithms:
```cpp
OutStreamBuf<...> outStreamBuf(...);
std::array<std::uint8_t, 128> data = {{.../* some data*/}};

std::copy(data.begin(), data.end(), std::back_inserter(outStreamBuf));
outStreamBuf.flush();
```
In the example above, [std::back_inserter](http://en.cppreference.com/w/cpp/iterator/back_inserter) requires a container to define `push_back()` member function:
```cpp
template <...>
class OutStreamBuf
{
public:
   // Wrap pushBack()
   void push_back(CharType ch)
   {
       pushBack(ch);
   }
   ...
};
```

There also may be a need to iterate over written, but still not flushed, characters and update some of them before the call to `flush()`. In other words the "Output Stream Buffer" must be treated as random access container:
```cpp
template <...>
class OutStreamBuf
{
public:
    typedef embxx::container::StaticQueue<CharType, BufSize> Buffer;
    typedef typename Buffer::Iterator Iterator;
    typedef typename Buffer::ConstIterator ConstIterator;
    typedef typename Buffer::ValueType ValueType;
    typedef typename Buffer::Reference Reference;
    typedef typename Buffer::ConstReference ConstReference;

    bool empty() const;
    void clear();
    void resize(std::size_t newSize);

    Iterator begin();
    Iterator end();

    ConstIterator begin() const;
    ConstIterator end() const;

    ConstIterator cbegin() const;
    ConstIterator cend() const;

    Reference operator[](std::size_t idx);
    ConstReference operator[](std::size_t idx) const;
   ...
};
```

As was mentioned earlier, the `OutStreamBuf` uses [Static (Fixed Size) Queue](../basic_needs/queue.md) as its internal buffer and any characters pushed beyond the capacity gets discarded. There must be a way to identify available capacity as well as request asynchronous notification via callback when requested capacity becomes available:
```cpp
template <typename TDriver,
          std::size_t TBufSize,
          typename TWaitHandler = 
              embxx::util::StaticFunction<void (const embxx::error::ErrorStatus&)> >
class OutStreamBuf
{
public:
    std::size_t availableCapacity() const;

    template <typename TFunc>
    void asyncWaitAvailableCapacity(
        std::size_t capacity,
        TFunc&& func)
    {
        if (capacity <= availableCapacity()) {
            ... // invoke callback via post() member function of Event Loop
        }
        waitAvailableCapacity_ = capacity;
        waitHandler_ = std::forward<TFunc>(func);

        // The Driver is writing some portion of flushed characters,
        // evaluate the capacity again when Driver reports completion.
    }
    
private:
    ...
    std::size_t waitAvailableCapacity_;
    WaitHandler waitHandler_;
};
```

Such "Output Stream Buffer" is already implemented in [embxx/io/OutStreamBuf.h](https://github.com/arobenko/embxx/blob/master/embxx/io/OutStreamBuf.h) file of [embxx](https://github.com/arobenko/embxx) library and documentation can be found [here](https://dl.dropboxusercontent.com/u/46999418/embxx/io_out_stream_buf_page.html).

The next stage would be defining the "Output Stream" class, which will allow printing of null terminated strings as well as various integral values. 
```cpp
template <typename TStreamBuf>
class OutStream
{
public:
    typedef typename TStreamBuf::CharType CharType;

    explicit OutStream(TStreamBuf& buf)
    : buf_(buf)
    {
    }

    OutStream(OutStream&) = delete;
    ~OutStream() = default;

    void flush()
    {
        buf_.flush();
    }

    OutStream& operator<<(const CharType* str)
    {
        while (*str != '\0') {
            buf_.pushBack(*str);
            ++str;
        }
        return *this;
    }

    OutStream& operator<<(char ch)
    {
        buf_.pushBack(ch);
        return *this;
    }

    OutStream& operator<<(std::uint8_t value)
    {
        // Cast std::uint8_t to unsigned and print.
        return (*this << static_cast<unsigned>(value));
    }

    OutStream& operator<<(std::int16_t value)
    {
        ... // Cast std::int16_t to int type and print.
        return *this;
    }

    OutStream& operator<<(std::uint16_t value)
    {
        // Cast std::uint16_t to unsigned and print
        return (*this << static_cast<std::uint32_t>(value));
    }

    OutStream& operator<<(std::int32_t value)
    {
        ... // Print signed value
        return *this;
    }

    OutStream& operator<<(std::uint32_t value)
    {
        ... // Print unsigned value
        return *this;
    }

    OutStream& operator<<(std::int64_t value)
    {
        ... // Print 64 bit signed value
        return *this
    }

    OutStream& operator<<(std::uint64_t value)
    {
        ... // Print 64 bit signed value
        return *this
    }

private:
    TStreamBuf& buf_;
};
```

We will also require the numeric base representation and manipulator. Unfortunately, usage of `std::oct`, `std::dec`or `std::hex` manipulators will require inclusion of standard library header [<ios>](http://en.cppreference.com/w/cpp/header/ios), which in turn includes other standard stream related headers, which define some static objects, which in turn are defined and instantiated in standard library. It contradicts our main goal of writing generic code that doesn't require standard library to be used. It is better to define such manipulators ourselves:
```cpp
enum Base
{
    bin, ///< Binary numeric base stream manipulator
    oct, ///< Octal numeric base stream manipulator
    dec, ///< Decimal numeric base stream manipulator
    hex, ///< Hexadecimal numeric base stream manipulator
    Base_NumOfBases ///< Must be last
};

template <typename TStreamBuf>
class OutStream
{
public:
    explicit OutStream(TStreamBuf& buf)
    : buf_(buf)
      base_(dec)
    {
    }

    OutStream& operator<<(Base value)
    {
        base_ = value;
        return *this
    }

private:
    TStreamBuf& buf_;
    Base base_;
};
```

The value of the numeric base representation must be taken into account when creating string representation of numeric values. The usage is very similar to standard:
```cpp
OutStream<...> stream;

stream << "var1=" << dec << var1 << "; var2=" << hex << var2 << '\n';
stream.flush();
```

It may be convenient to support a little bit of formatting, such as specifying minimal width of the output as well as fill character:
```cpp
class WidthManip : public ValueManipBase<std::size_t>
{
public:
    WidthManip(std::size_t value) : value_(value) {}
    std::size_t value() const { return value_;}
private:
    std::size_t value_;
};

inline
WidthManip setw(std::size_t value)
{
    return WidthManip(value);
}

template <typename T>
class FillManip
{
public:
    FillManip(T value) : value_(value) {}
    T value() const { return value_;}
private:
    T value_;
};

template <typename T>
inline
FillManip<T> setfill(T value)
{
    return FillManip<T>(value);
}

template <typename TStreamBuf>
class OutStream
{
public:
    explicit OutStream(TStreamBuf& buf)
    : buf_(buf)
      base_(dec),
      width_(0),
      fill_(static_cast<CharType>(' ');
    {
    }

    OutStream& operator<<(WidthManip manip)
    {
        width_ = manip.value();
        return *this;
    }

    template <typename T>
    OutStream& operator<<(details::FillManip<T> manip)
    {
        fill_ = static_cast<CharType>(manip.value());
        return *this;
    }

private:
    TStreamBuf& buf_;
    Base base_;
    std::size_t width_;
    CharType fill_;
};
```

The usage is very similar to the base manipulator:
```cpp
OutStream<...> stream;

stream << "var1=" << dec << setw(4) << var1 << "; var2=" << hex 
       << setfill('0') << var2 << '\n';
stream.flush();
```

Another useful manipulator is adding '\n' at the end as well as calling `flush()`, just like `std::endl` does when using standard output streams:
```cpp
enum Endl
{
    endl ///< End of line stream manipulator
};

template <typename TStreamBuf>
class OutStream
{
public:

    OutStream& operator<<(Endl manip)
    {
        static_cast<void>(manip);
        buf_.pushBack(static_cast<CharType>('\n');
        flush();
        return *this;
    }


private:
    ...
};

```

Then usage example may be changed to:
```cpp
OutStream<...> stream;

stream << "var1=" << dec << setw(4) << var1 << "; var2=" << hex 
       << setfill('0') << var2 << endl;
```

**To summarise**: The "Output Stream" object converts given integer value into the printable characters and uses `pushBack()` member function of "Output Stream Buffer" to pass these characters further. The request to `flush()` is also passed on. When "Output Stream Buffer" receives a request to flush internal buffer it activates the "Character" **Driver**, which it turn uses "UART" **Device** to write characters to serial interface one by one. As the result of such cooperation, the "printing" statement is very quick, there is no wait for all the characters to be written before the function returns, like it is usually done with `printf()`. All the characters are written at the background using interrupts, while the main thread of the application continues its execution without stalling.

Such "Output Stream" is already implemented in [embxx/io/OutStream.h](https://github.com/arobenko/embxx/blob/master/embxx/io/OutStream.h) file of [embxx](https://github.com/arobenko/embxx) library and documentation can be found [here](https://dl.dropboxusercontent.com/u/46999418/embxx/io_out_stream_page.html).

### Logging

In general, debug logging should be under conditional compilation, for example only in **DEBUG** mode, while the printing code is excluded when compiling in **RELEASE** mode.
```cpp
#ifndef NDEBUG
    stream << "Some info massage" << endl;
#endif
```

Sometimes there is a need to easily change the amount of debug messages being printed. For that purpose, the concept of logging levels is widely used:
```cpp
namespace log
{

enum Level
{
    Trace, ///< Use for tracing enter to and exit from functions.
    Debug, ///< Use for debugging information.
    Info, ///< Use for general informative output.
    Warning, ///< Use for warning about potential dangers.
    Error, ///< Use to report execution errors.
    NumOfLogLevels ///< Number of log levels, must be last
};

}  // namespace log
```

The logging statement becomes a macro:
```cpp
const auto MinLogLevel = log::Info;

#define LOG(stream__, level__, output__) \
    do { \
        if (MinLevel <= (level__)) { \
            (stream__).stream() << output__; \
        } \
    } while (false)
```
In this case all the logging attempts for level below `log::Info` get optimised away by the compiler, because the `if` statement known to evaluate to `false` at compile time:
```cpp
LOG(stream, log::Debug, "This message is not printed." << endl);
LOG(stream, log::Info, "This message IS printed." << endl);
LOG(stream, log::Warning, "This message IS printed also." << endl);
```

It would be nice to be able to add some automatic formatting to the logged statements, such as printing the log level and/or adding '\n' and flushing at the end. For example, the code below
```cpp
LOG(stream, log::Debug, "This is DEBUG message.");
LOG(stream, log::Info, "This is INFO message.");
LOG(stream, log::Warning, "This is WARNING message.");
```
to produce the following output
```
[DEBUG]: This is DEBUG message.
[INFO]: This is INFO message.
[WARNING]: This is WARNING message.
```
with '\n' character and call to `flush()` at the end.

It is easy to achieve when using some kind of wrapper logging class around the output stream as well as relevant formatters.
For example:
```cpp
template <log::Level TLevel, typename TStream>
class StreamLogger
{
public:

    typedef TStream Stream;

    static const log::Level MinLevel = TLevel;

    explicit StreamLogger(Stream& outStream)
      : outStream_(outStream)
    {
    }

    Stream& stream()
    {
        return outStream_;
    }

    // Begin output. This function is called before requested 
    // output is redirected to stream. It does nothing.
    void begin(log::Level level)
    {
        static_cast<void>(level);
    }

    // End output. This function is called after requested 
    // output is redirected to stream. It does nothing.
    void end(log::Level level)
    {
        static_cast<void>(level);
    }

private:
    Stream& outStream_;
};
```

The logging macro will look like this:
```cpp
#define SLOG(log__, level__, output__) \
    do { \
        if ((log__).MinLevel <= (level__)) { \
            (log__).begin(level__); \
            (log__).stream() << output__; \
            (log__).end(level__); \
        } \
    } while (false)
```

A formatter can be defined by exposing the same interface, but wraps the original `StreamLogger` or another formatter. For example let's define formatter that calls `flush()` member function of the stream when output is complete:
```cpp
template <typename TNextLayer>
class StreamFlushSuffixer
{
public:

    // Constructor, forwards all the other parameters to the constructor
    // of the next layer.
    template<typename... TParams>
    StreamFlushSuffixer(TParams&&... params)
      : nextLavel_(std::forward<TParams>(params)...)
    {
    }

    Stream& stream()
    {
        return nextLavel_.stream();
    }

    void begin(log::Level level)
    {
        nextLavel_.begin(level);
    }

    void end(log::Level level)
    {
        nextLavel_.end(level);
        stream().flush();
    }

private:
    TNextLavel nextLavel_;
};

```

The definition of such logger would be:
```cpp
typedef ... OutStream; // type of the output stream
typedef 
    StreamFlushSuffixer<
        StreamLogger<
            log::Debug,
            OutStream
        >
    > Log;
```

The same `SLOG()` macro will work for this logger with extra formatting:
```cpp
OutStream stream(... /* construction params */);
Log log(stream);
SLOG(log, log::Debug, "This is DEBUG message.\n");
```

Let's also add a formatter that capable of printing any value (and '\n' in particular) at the end of the output. 
```cpp
template <typename T, typename TNextLayer>
class StreamableValueSuffixer
{
public:

    template<typename... TParams>
    explicit StreamableValueSuffixer(T&& value, TParams&&... params)
      : value_(std::forward<T>(value)),
        nextLevel_(std::forward<TParams>(params)...)
    {
    }

    Stream& stream()
    {
        return nextLavel_.stream();
    }

    void begin(log::Level level)
    {
        nextLavel_.begin(level);
    }

    void end(log::Level level)
    {
        nextLavel_.end(level);
        stream() << value_;
    }


private:
    T value_;
    TNextLavel nextLavel_;
};
```

The definition of the logger that adds '\n' character and then calls `flush()` member function of the underlying stream would be:
```cpp
typedef embxx::io::OutStream<...> OutStream;
typedef 
    StreamFlushSuffixer<
        StreamableValueSuffixer<
            char,
            StreamLogger<
                log::Debug,
                OutStream
            >
        >
    > Log;
```

While the construction will require to specify the character which is going to be printed at the end, but before call to `flush()`.
```cpp
OutStream stream(...);
Log log('\n', stream);
SLOG(log, log::Debug, "This is DEBUG message.");
```

As the last formatter, let's do the one that prefixes the output with log level information:
```cpp
template <typename TNextLayer>
class LevelStringPrefixer
{
public:
    template<typename... TParams>
    LevelStringPrefixer(TParams&&... params);
      : next_value(std::forward<TParams>(params)...)
    {
    }

    Stream& stream()
    {
        return nextLavel_.stream();
    }

    void begin(Level level)
    {
        static const char* const Strings[NumOfLogLevels] = {
            "[TRACE] ",
            "[DEBUG] ",
            "[INFO] ",
            "[WARNING] ",
            "[ERROR] "
        };

        if ((level < NumOfLogLevels) && (Strings[level] != nullptr)) {
            stream() << Strings[level];
        }

        nextLavel_.begin(level);
    }

    void end(log::Level level)
    {
        nextLavel_.end(level);
    }

private:
    TNextLavel nextLavel_;
};
```

The definition of the logger that prints such a prefix at the beginning and '\n' at the end together with call to `flush()` would be:
```cpp
typedef 
    StreamFlushSuffixer<
        StreamableValueSuffixer<
            char,
            LevelStringPrefixer<
                StreamLogger<
                    log::Debug,
                    OutStream
                >
            >
        >
    > Log;
```

Such `StreamLogger` together with multiple formatters is already implemented in [embxx/util/StreamLogger.h](https://github.com/arobenko/embxx/blob/master/embxx/util/StreamLogger.h) file of [embxx](https://github.com/arobenko/embxx) library and documented [here](https://dl.dropboxusercontent.com/u/46999418/embxx/util_stream_logger_page.html).

### Logging Application

The [app_uart1_logging](https://github.com/arobenko/embxx_on_rpi/tree/master/src/app/app_uart1_logging) application in [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project implements logging of simple counter that gets incremented once a second:
```cpp
namespace log = embxx::util::log;
template <typename TLog, typename TTimer>
void performLog(TLog& log, TTimer& timer, std::size_t& counter)
{
    ++counter;

    SLOG(log, log::Info,
        "Logging output: counter = " <<
        embxx::io::dec << counter <<
        " (0x" << embxx::io::hex << counter << ")");

    // Perform next logging after a timeout
    static const auto LoggingWaitPeriod = std::chrono::seconds(1);
    timer.asyncWait(
        LoggingWaitPeriod,
        [&](const embxx::error::ErrorStatus& es)
        {
            GASSERT(!es);
            static_cast<void>(es);
            performLog(log, timer, counter);
        });
}

int main() {
    auto& system = System::instance();
    auto& log = system.log();

    // Configure UART
    auto& uart = system.uart();
    uart.configBaud(115200);
    uart.setWriteEnabled(true);

    // Timer allocation
    auto timer = system.timerMgr().allocTimer();
    GASSERT(timer.isValid());

    // Start logging
    std::size_t counter = 0;
    performLog(log, timer, counter);

    // Run event loop
    device::interrupt::enable();
    auto& el = system.eventLoop();
    el.run();

    GASSERT(0); // Mustn't exit
    return 0;
}
```

The [System.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/app/app_uart1_logging/System.h) file defines the whole output stack:
```cpp
class System
{
public:
    static const std::size_t EventLoopSpaceSize = 1024;
    typedef embxx::util::EventLoop<
        EventLoopSpaceSize,
        device::InterruptLock,
        device::WaitCond> EventLoop;

    // Devices
    typedef device::Uart1<InterruptMgr> Uart;
    ...

    // Drivers
    struct CharacterTraits
    {
        typedef std::nullptr_t ReadHandler;
        typedef embxx::util::StaticFunction<
            void(const embxx::error::ErrorStatus&, std::size_t)> WriteHandler;
        typedef std::nullptr_t ReadUntilPred;
        static const std::size_t ReadQueueSize = 0;
        static const std::size_t WriteQueueSize = 1;
    };
    typedef embxx::driver::Character<
        Uart, EventLoop, CharacterTraits> UartDriver;
    ...

    // Components
    static const std::size_t OutStreamBufSize = 1024;
    typedef embxx::io::OutStreamBuf<
        UartDriver, OutStreamBufSize> OutStreamBuf;

    typedef embxx::io::OutStream<OutStreamBuf> OutStream;
    typedef embxx::util::log::StreamFlushSuffixer<
            embxx::util::log::StreamableValueSuffixer<
                const OutStream::CharType*,
                embxx::util::log::LevelStringPrefixer<
                    embxx::util::StreamLogger<
                        embxx::util::log::Debug,
                        OutStream
                    >
                >
            >
        > Log;

    ...
private:

    EventLoop el_;

    // Devices
    Uart uart_;
    ...

    // Drivers
    UartDriver uartDriver_;
    ...

    // Components
    OutStreamBuf buf_;
    OutStream stream_;
    Log log_;
    ...
};

```

This application will produce the following output to the UART interface with new line appearing every second:
```
[INFO] Logging output: counter = 1 (0x1)
[INFO] Logging output: counter = 2 (0x2)
[INFO] Logging output: counter = 3 (0x3)
...
```

### Buffered Input

In many systems the UART interfaces are also used to communicate between various microcontrollers on the same board or with external devices. When there are incoming messages, the characters must be stored in some buffer before they can be processed by some **Component**. Just like we had "Output Stream Buffer" for buffering outgoing characters, we must have "Input Stream Buffer" for buffering incoming ones.

It must obviously have an access to the Character **Driver** and will probably have a circular buffer to store incoming characters.
```cpp
template <typename TDriver, std::size_t TBufSize>
class InStreamBuf
{
public:
    typedef typename TDriver::CharType CharType;
    typedef embxx::container::StaticQueue<CharType, TBufSize> Buffer;

    explicit 
    InStreamBuf(TDriver& driver)
      : driver_(driver)
    {
    }

private:
    TDriver& driver_;
    Buffer buf_;
};
```

The **Driver** won't perform any read operations unless it is explicitly requested to do so with its `asyncRead()` member function. Sometimes, there is a need to keep characters flowing in and being stored in the buffer, even when the **Component** responsible for processing them is not ready. In order to make this happen, the "Input Stream Buffer" must be responsible for constantly requesting the **Driver** to perform asynchronous read while providing space where these characters are going to be stored. 
```cpp
template <typename TDriver, std::size_t TBufSize>
class InStreamBuf
{
public:
    // Start data accumulation in the internal buffer.
    void start();

    // Stop data accumulation in the internal buffer.
    void stop();

    // Inquire whether characters are being accumulated.
    bool isRunning() const;
};
```

Most of the times the responsible **Component** will require some number of characters to be accumulated before their processing can be started. There is a need to provide asynchronous notification callback request when appropriate number of characters becomes available. The callback must be stored in the internal data structures of the "Input Stream Buffer" and invoked when needed. Due to the latter being developed as a generic class, there is a need to provide callback storage type as a template parameter.
```cpp
template <typename TDriver, std::size_t TBufSize, typename TWaitHandler>
class InStreamBuf
{
public:

    template <typename TFunc>
    void asyncWaitDataAvailable(std::size_t reqSize, TFunc&& func)
    {
        callback_ = std::forward<TFunc>(func)
        ...
    }

private:
    TWaitHandler callback_;
};
```

Once the required number of characters is accumulated, the **Component** must be able to access and process them. It means that "Input Stream Buffer" must also be a container with random access iterators.
```cpp
template <typename TDriver, std::size_t TBufSize, typename TWaitHandler>
class InStreamBuf
{
public:
    typedef typename Buffer::ConstIterator ConstIterator;
    typedef ConstIterator const_iterator;
    typedef typename Buffer::ValueType ValueType;
    typedef ValueType value_type;
    typedef typename Buffer::ConstReference ConstReference;
    typedef ConstReference const_reference;

    // Get size of available for read data.
    std::size_t size() const;

    // Check whether number of available characters is 0.
    bool empty() const;

    //Get full capacity of the buffer.
    constexpr std::size_t fullCapacity() const;

    ConstIterator begin() const;
    ConstIterator end() const;
    ConstIterator cbegin() const;
    ConstIterator cend() const;
    ConstReference operator[](std::size_t idx) const;
};
```

Please note, that all the access to the characters are done using const iterator. It means we do not allow external and uncontrolled update of the characters inside of the buffer.

When the characters inside the buffer got processed and aren't needed any more, they need to be discarded to free the space inside the buffer for new ones to come.

```cpp
template <typename TDriver, std::size_t TBufSize, typename TWaitHandler>
class InStreamBuf
{
public:
    // Consume part or the whole buffer of the available data for read.
    void consume(std::size_t consumeSize = size());
};
```

### Morse Code Application

The [app_uart1_morse](https://github.com/arobenko/embxx_on_rpi/tree/master/src/app/app_uart1_morse) application in [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project implements buffering of incoming characters in the "Input Stream Buffer" and uses the [Morse Code](http://en.wikipedia.org/wiki/Morse_code) method to display them by flashing the on-board led.

First of all there is a need to have an access to the led to flash, input buffer to store the incoming characters and timer manager to allocate a timer to measure timeouts.
```cpp
template <typename TLed, typename TInBuf, typename TTimerMgr>
class Morse
{
public:
    typedef TLed Led;
    typedef TInBuf InBuf;
    typedef TTimerMgr TimerMgr;
    typedef typename TimerMgr::Timer Timer;

    Morse(Led& led, InBuf& buf, TimerMgr& timerMgr)
      : led_(led),
        buf_(buf),
        timer_(timerMgr.allocTimer())
    {
        GASSERT(timer_.isValid());
    }

    ~Morse() = default;

private:
    Led& led_;
    InBuf& buf_;
    Timer timer_;
};
```

Second, there is a need to define a Morse code sequences in terms of dots and dashes duration as well as mapping an incoming character to the respective sequence.

```cpp
template <...>
class Morse
{
public:
    typedef typename InBuf::CharType CharType;
    ...
private:
    typedef unsigned Duration;
    static const Duration Dot = 200;
    static const Duration Dash = Dot * 3;
    static const Duration End = 0;
    static const Duration Spacing = Dot;
    static const Duration InterSpacing = Spacing * 2;

    const Duration* getLettersSeq(CharType ch) const
    {
        static const Duration Seq_A[] = {Dot, Dash, End};
        static const Duration Seq_B[] = {Dash, Dot, Dot, Dot, End};
        ...
        static const Duration Seq_Z[] = {Dash, Dash, Dot, Dot, End};

        static const Duration Seq_0[] = {
            Dash, Dash, Dash, Dash, Dash, End};
        static const Duration Seq_1[] = {
            Dot, Dash, Dash, Dash, Dash, End};
        ...
        static const Duration Seq_9[] = {
            Dash, Dash, Dash, Dash, Dot, End};

        static const Duration* Letters[] = {
            Seq_A,
            Seq_B,
            ...
            Seq_Z
        };

        static const Duration* Numbers[] = {
            Seq_0,
            ...
            Seq_9
        };


        if ((static_cast<CharType>('A') <= ch) &&
            (ch <= static_cast<CharType>('Z'))) {
            return Letters[ch - 'A'];
        }

        if ((static_cast<CharType>('a') <= ch) &&
            (ch <= static_cast<CharType>('z'))) {
            return Letters[ch - 'a'];
        }

        if ((static_cast<CharType>('0') <= ch) &&
            (ch <= static_cast<CharType>('9'))) {
            return Numbers[ch - '0'];
        }

        return nullptr;
    }
};
```

Now, the code that is responsible to flash a led is quite simple:
```cpp
template <...>
class Morse
{
public:

    void start()
    {
        buf_.start();
        nextLetter();
    }

private:
    void nextLetter()
    {
        buf_.asyncWaitDataAvailable(
            1U,
            [this](const embxx::error::ErrorStatus& es)
            {
                if (es) {
                    GASSERT(buf_.empty());
                    nextLetter();
                    return;
                }

                GASSERT(!buf_.empty());
                auto ch = buf_[0];
                buf_.consume(1U);

                auto* seq = getLettersSeq(ch);
                if (seq == nullptr) {
                    nextLetter();
                    return;
                }

                nextSyllable(seq);
            });
    }

    void nextSyllable(const Duration* seq)
    {
        GASSERT(seq != nullptr);
        GASSERT(*seq != End);

        auto duration = *seq;
        ++seq;

        led_.on();
        timer_.asyncWait(
            std::chrono::milliseconds(duration),
            [this, seq](const embxx::error::ErrorStatus& es)
            {
                static_cast<void>(es);
                GASSERT(!es);

                led_.off();

                if (*seq != End) {
                    timer_.asyncWait(
                        std::chrono::milliseconds(Duration(Spacing)),
                        [this, seq](const embxx::error::ErrorStatus& es)
                        {
                            static_cast<void>(es);
                            GASSERT(!es);
                            nextSyllable(seq);
                        });
                    return;
                }

                timer_.asyncWait(
                    std::chrono::milliseconds(Duration(InterSpacing)),
                    [this](const embxx::error::ErrorStatus& es)
                    {
                        static_cast<void>(es);
                        GASSERT(!es);
                        nextLetter();
                    });
            });
    }

};
```

The `nextLetter()` member function waits until one character becomes available in the buffer, then maps it to the sequence and removes it from the buffer. If the mapping exists it calls the `nextSyllable()` member function to start the flashing sequence. The function activates the led and waits the relevant amount of time, based on the provided dot or dash duration. After the timeout, the led goes off and new wait is activated. However if the end of sequence is reached, the wait will be of `InterSpacing` duration and `nextLetter()` member function will be called again, otherwise the wait will be of `Spacing` duration and `nextSyllable()` will be called again to activate the led and wait for the next period in the sequence.

### Summary

After this quite a significant effort we've created a full generic stack to perform asynchronous input/output operations over serial interface, such as UART. It may be reused in multiple independent projects while providing platform specific low level device control object at the bottom of this stack.
