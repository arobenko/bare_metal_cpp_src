## SPI

[SPI](http://en.wikipedia.org/wiki/Serial_Peripheral_Interface_Bus) is also quite popular serial communication interface. It is very similar to [I2C](i2c.md) in terms of using it the [Device-Driver-Component](../basic_concepts/device_driver_component.md) model described in this book. The main differences are:

1. SPI uses "chip select" identification method instead of "address" of the peripheral.
1. SPI is a double direction link - there are always read and write operations that are executed in parallel (instead of only read or only write).

The "chip select" slave identefication will require the same "**ID Adaptor**" that was used for [I2C](i2c.md) integration.

Just like with [I2C](i2c.md), the [SPI](http://en.wikipedia.org/wiki/Serial_Peripheral_Interface_Bus) is a multi-slave bus. It allows connection of multiple independent devices to the same MISO/MOSI/CLK lines of the SPI interface. It means there is a need for the same "**Operations Queue**" that was used for [I2C](i2c.md) integration. Due to the fact that SPI is a double direction link, the "**Operations Queue**" must be able to forward, say, read operation request to the actual **Device** even if "write" operation to the same slave device is already in progress. 

It means that the objects' usage map is exactly the same as with [I2C](i2c.md).

![Image: Using Op Queue](../image/op_queue.png)

All the intermediate layers (Character **Driver**, ID Adaptor, Operations Queue) in the map above must allow issuing read and write operations at the same time. It becomes a responsibility of the product specific **Component** to be aware what kind of the **Device** is used and not to issue these requests in parallel if the actual **Device** (such as I2C) doesn't support it.

### SPI Device

Based on the information above, the platform specific SPI control **Device** object must provide and implement exactly the same interface as [I2C](i2c.md) **Device**:
```cpp
class SpiDevice
{
public:
    // Single character type
    typedef std::uint8_t CharType;

    // ID type - chip select index
    typedef unsigned DeviceIdType; 

    // Context types
    typedef embxx::device::context::EventLoop EventLoopContext;
    typedef embxx::device::context::Interrupt InterruptContext;

    // Set various interrupt handlers
    template <typename TFunc>
    void setCanReadHandler(TFunc&& func);

    template <typename TFunc>
    void setCanWriteHandler(TFunc&& func);

    template <typename TFunc>
    void setReadCompleteHandler(TFunc&& func);

    template <typename TFunc>
    void setWriteCompleteHandler(TFunc&& func);

    // Start read for both contexts.
    void startRead(DeviceIdType chipSelect, std::size_t length, EventLoopContext);
    void startRead(DeviceIdType chipSelect, std::size_t length, InterruptContext);

    // Cancel read for both contexts.
    bool cancelRead(EventLoopContext);
    bool cancelRead(InterruptContext);

    // Start write for both contexts.
    void startWrite(DeviceIdType chipSelect, std::size_t length, EventLoopContext);
    void startWrite(DeviceIdType chipSelect, std::size_t length, InterruptContext);
        TContext context);

    // Cancel write for both contexts.
    bool cancelWrite(EventLoopContext);
    bool cancelWrite(InterruptContext);

    // Suspend/Resume
    bool suspend(EventLoopContext);
    void resume(EventLoopContext);

    // Helper functions to manage read/write during the interrupt
    bool canRead(InterruptContext);
    bool canWrite(InterruptContext);
    CharType read(InterruptContext);
    void write(CharType value, InterruptContext);
};
```

Such device to control **SPI0** interface on RaspberryPi platform is implemented in [src/device/Spi0.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/Spi0.h) file of [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project.

### Other Nuances

SPI is quite often used  with external persistent storage, such as SD card. Such devices may have some significant delays between the block write operation on the `MOSI` line and the time they send an acknowledgement about operation completion on the `MISO` line. The SPI **Device** must constantly read the incoming bytes until the expected `ACK`/`NACK` byte is received without de-asserting the `CS` (chip select). If the **Component**, responsible for managing SPI flash memory, issues only single "read" operation to wait for such an acknowledgement, the provided buffer may get full before the required byte is received. In this case the SPI control **Device** object is not aware that the new "read" request may follow and has to de-assert the `CS`, which is undesireble.

In order to solve this problem, the Character **Driver** described in [UART](uart.md) chapter must be extended to support issuing multiple read/write operations at the same time. Such extension is based on the values of `ReadQueueSize`/`WriteQueueSize` in the provided `Traits` class. These values indicate maximal number of simultaneous read/write operations that may be issued to the **Driver**. The responsible **Component**, in turn, must perform 2 or 3 "read until" operations at the same time to wait for the expected response. Once the first buffer is full, the **Driver** will post the **Component**'s callback object for execution in the event loop context, while calling `startRead()` member function of the **Device** for the next pending "read until" operation still in interrupt context to fill the second buffer. The **Device** is responsible to continue its read operation without de-asserting the `CS` line. While the second buffer being filled, the **Component** has enough time to identify that there is no response in the filled buffer and re-issue the "read until" request to the **Driver** while reusing the same buffer. This circle of "read until" requests must continue until expected response is encountered or until operation timeout, which is measured independently by the asynchronous wait request to the [Timer](timer.md). It is up to the responsible **Component** object to manage the operations to the Character **Driver** as well as the Timer in event loop context and cancel one upon execution of callback from another.

### External Storage 

As was mentioned in previous section, SPI is often used with external persistent storage, such as SD card. In order to properly support it, there must be some kind of `SpiFlash` management **Component**, that is responsible to implement proper [communication protocol](https://www.sdcard.org/downloads/pls/simplified_specs/part1_410.pdf) while providing necessary public interface. The minimal required interface will have to be able to:
1. Asynchronously initialise the device.
1. Asynchronously read block of data.
1. Asynchronously write block of data.

Once such **Component** is implemented and tested, the next stage would be implementing proper file system (FAT32) management **Component**, using the asynchronous functions of the former. It will allow processing time consuming file system reads and writes while still allowing processing of all other events without creating any performance bottlenecks and without requiring any complex independent task scheduling.

