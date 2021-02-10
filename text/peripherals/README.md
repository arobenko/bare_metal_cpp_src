# Peripherals

It this chapter I will describe and give multiple examples of how to drive and control multiple hardware peripherals while using [Device-Driver-Component](../basic_concepts/device_driver_component.md) model in conjunction with [Event Loop](../basic_concepts/event_loop.md).

All the generic, platform independent code provided here is implemented as part of [embxx](https://github.com/arobenko/embxx) library while platform (Raspberry Pi) specific code is taken from [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project.

All the platform specific peripheral control classes reside in [src/device](https://github.com/arobenko/embxx_on_rpi/tree/master/src/device) directory.

The [src/app](https://github.com/arobenko/embxx_on_rpi/tree/master/src/app) directory contains several simple applications, such as flashing the led or responding to button presses.

There are also common **Component** classes shared between the applications. They reside in [src/component](https://github.com/arobenko/embxx_on_rpi/tree/master/src/component) directory.

In order to compile all the applications please follow the instructions described in [Contents of This Document](../overview/contents.md).

## Function Configuration

In ARM platform every pin needs to be configured as either gpio input, gpio output or having one of several alternative functions the microcontroller supports. The `device::Function` class defined in [src/device/Function.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/Function.h) and [src/device/Function.cpp](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/Function.cpp) implements simple interface which allows every **Device** class configure the pins it uses.
```cpp
class Function
{
public:
    enum class FuncSel {
        Input,  // b000
        Output, // b001
        Alt5,   // b010
        Alt4,   // b011
        Alt0,   // b100
        Alt1,   // b101
        Alt2,   // b110
        Alt3    // b111
    };

    typedef unsigned PinIdxType;

    static const std::size_t NumOfLines = 54;

    void configure(PinIdxType idx, FuncSel sel);
};

```

Every implemented **Device** class will receive reference to `Function` object in its constructor and will have to use it to configure the pins as required.

## Interrupts Management

There is one more componenet that every **Device** will use. It's `device::InterruptMgr` defined in [src/device/InterruptMgr.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/InterruptMgr.h). The main responsibility of the object of this class is to control global level interrupts, register interrupt handlers from various **Device**s and invoke the appropriate handler when interrupt occurs.

The interface of the `device::InterruptMgr` is defined as following:
```cpp
template <typename THandler = embxx::util::StaticFunction<void ()> >
class InterruptMgr
{
public:
    typedef THandler HandlerFunc;
    enum IrqId {
        IrqId_Timer,
        IrqId_AuxInt,
        IrqId_Gpio1,
        IrqId_Gpio2,
        IrqId_Gpio3,
        IrqId_Gpio4,
        IrqId_I2C,
        IrqId_SPI,
        IrqId_NumOfIds // Must be last
    };

    InterruptMgr();

    template <typename TFunc>
    void registerHandler(IrqId id, TFunc&& handler);

    void enableInterrupt(IrqId id);

    void disableInterrupt(IrqId id);

    void handleInterrupt();

private:
    typedef std::uint32_t EntryType;

    struct IrqInfo {
        ... // Contains interrupt related information 
            // per single IrqId
    };

    typedef std::array<IrqInfo, IrqId_NumOfIds> IrqsArray;

    IrqsArray irqs_;
};
```

Every **Driver** will use `registerHandler()` member function to register its member function as the handler for its `IrqId`. The `enableInterrupt()` and `disableInterrupt()` are also used by the **Device** objects to control their interrupts on global level.

In order to use the **Interrupt Manager** described above every application has to implement proper interrupt handler that will retrieve the reference to `device::InterruptMgr` object (via global/static variables) and invoke its `handleInterrupt()` function, which in turn check the appropriate status register(s) and invoke registered handler(s). Please note, that the handler will be executed in interrupt context.

The code will look something like this:
```cpp
extern "C"
void interruptHandler()
{
    System::instance().interruptMgr().handleInterrupt();
}
```

There may also be a need to enable/disable all the interrupts by toggling `i` flag in `CPS` register. The same [src/device/InterruptMgr.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/InterruptMgr.h) file provides two function for this purpose:
```cpp
namespace device
{

namespace interrupt
{

inline
void enable()
{
    __asm volatile("cpsie i");
}

inline
void disable()
{
    __asm volatile("cpsid i");
}

}  // namespace interrupt

}  // namespace device
```
