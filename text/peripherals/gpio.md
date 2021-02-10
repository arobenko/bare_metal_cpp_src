## GPIO

In many cases, the GPIO input doesn't need to be processed at the same time the interrupt has occured. It can easilily be scheduled for execution in event loop (non-interrupt) context using [Device-Driver-Component](../basic_concepts/device_driver_component.md) model.

According to what was written in [Device-Driver-Component](../basic_concepts/device_driver_component.md) chapter and to what we've seen so far, the **Component** provides a callback object together with the asynchronous operation request. The callback is executed only **once** when the operation is compete, canceled or terminated due to some error. If the operation needs to be repeated, another asynchronous operation needs to be issued to the **Driver** while providing another callback object to be called on operation completion.

The need for GPIO input handling is a bit different though. The line may change its value multiple times between the reporting of the event to the **Component** and the latter re-requesting asynchronous wait on value change. The **Driver** must preserve the callback object, provided by the **Component**, and invoke it every time the GPIO input value changes until the **Component** cancels the operation.

Let's go through all the stages in more detail.

### Configuration

![Image: GPIO register handler](../image/gpio_config.png)

The **Device** must provide a callback object to handle GPIO interrupts on all the requested input lines.

The hardware must also be configured properly: input/output lines, the interrupts on the rising/falling edges, etc.
Such configuration is platform/product specific and is not part of the generic [Device-Driver-Component](../basic_concepts/device_driver_component.md) model presented in this book. Hence, the product specific **Component** must get an access to the device object and configure it as needed.

### Start Continuous Asynchronous Read Operation

The **Driver** must be able to support multiple asynchronous read operations on different inputs. It means that it must protect an access to the internal data structures by requesting the **Device** to suspend the callback invocation (i.e. disable interrupts). Also to follow the pattern we used so far, there must be a request to start or enable the **Device**'s operation on the first read request and cancel or disable it on the last.

![Image: GPIO read](../image/gpio_read.png)

The reader may notice that on the first `asyncReadCont()` request, the **Driver** issued `suspend()` request to the **Device** and got `false` in return. It means that the **Device**'s monitoring of the GPIO inputs hasn't been started yet. That's the reason for the following call to `enable()`. On the second `asyncReadCont()` request the call to `suspend()` returned true which was followed by the `resume()` later.


### Reporting GPIO Input Event

Now, every time the relevant GPIO interrupt occurs, the **Driver**'s handler is invoked in interrupt mode context. It is responsible to schedule the execution of **Component**'s handler in event loop (non-interrupt) context.

![Image: GPIO interrupt report](../image/gpio_int_report.png)

### Cancel Continuous Read Operation.

When the there is no need to monitor some input any more, the **Component** may request the **Driver** to cancel the continuous asynchronous read operation. In case of last recorded asynchronous read operation being canceled, the **Driver** is responsible to let the **Device** know that no more GPIO interrupts are needed: 

![Image: GPIO cancel read](../image/gpio_cancel.png)

### GPIO Device

Based on the information above, the platform specific GPIO control **Device** object must provide the following public interface:
1. Define pin identification type.
```cpp
typedef unsigned PinIdType;
```
1. Function to provide a callback object to be called when interrupt occurs. The callback parameters must provide an information of pin as well as final input value that caused the interrupt. The callback object must implement the following signature: "void (PinIdType, bool)" where the first parameter is pin and second parameter is input value.
```cpp
template <typename TFunc>
void setHandler(TFunc&& func);
```
1. Function to start / enable the GPIO input monitoring.
```cpp
void start(embxx::device::context::EventLoop context);
```
1. Function to cancel / disable the GPIO input monitoring.
```cpp
bool cancel(embxx::device::context::EventLoop context);
```
1. Function to enable/disable gpio interrupts for single pin.
```cpp
void setEnabled(
    PinIdType pin, 
    bool enabled, 
    embxx::device::context::EventLoop context);
```
1. Function to suspend invocation of callback in interrupt mode, i.e. disable gpio interrupts.
```cpp
bool suspend(embxx::device::context::EventLoop context);
```
1. Function to resume suspended invocation of callback in interrupt mode, i.e. enable gpio interrupts.
```cpp
void resume(embxx::device::context::EventLoop context);
```

Such GPIO control **Device** class for RaspberryPi platform is implemented in [src/device/Gpio.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/Gpio.h) file of [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project. 

### GPIO Driver

First of all, we will need references to **Device** as well as **[Event Loop](../basic_concepts/event_loop.md)** objects:
```cpp
template <typename TDevice, typename TEventLoop>
class MyGpioDriver
{
public:
    // During the construction store references to Device
    // and Event Loop objects.
    MyGpioDriver(TDevice& device, TEventLoop& el)
      : device_(device),
        el_(el)
    {
        // Register appropriate interrupt callbacks with device
        device_.setHandler(...);
    }

    ...

private:

    TDevice& device_;
    TEventLoop& el_;
};
```

The **Driver** must also provide an ability to perform and cancel continuous asynchronous read operations for multiple pins:
```cpp
template <typename TDevice, typename TEventLoop>
class MyGpioDriver
{
public:
    typedef typename TDevice::PinIdType PinIdType;

    template <typename TFunc>
    void asyncReadCont(PinIdType id, TFunc&& func) { ... }

    bool cancelReadCont(PinIdType id) { ... }
};
```

Like with any asynchronous operation so far the callback must receive status information as its first parameter and probably the value of the input as the second one. When the operation canceled with `cancelReadCont()`, the callback must be invoked one last time with status specifying that operation was `Aborted`.

The **Driver** is supposed to be a generic piece of code that can be reused in multiple independent products, including ones without dynamic memory allocation and/or exceptions. It means that the **Driver** class must receive maximum number of the pins it is going to support and type of the callback storage.
```cpp
template <typename TDevice, 
          typename TEventLoop,
          std::size_t TNumOfLines,
          typename THandler =
              embxx::util::StaticFunction<void (const embxx::error::ErrorStatus&, bool)> >
class MyGpioDriver
{
public:
    template <typename TFunc>
    void asyncReadCont(PinIdType id, TFunc&& func) 
    {
        ...
        auto* node = ...; // Locate or allocate appropriate node
        node->id_ = id;
        node->handler_ = std::forward<TFunc>(func);
        ...
    }        

private:
    struct Node
    {
        Node() : id_(PinIdType()) {}

        PinIdType id_;
        THandler handler_;
    };

    typedef std::array<Node, TNumOfLines> Infos;

    Infos infos_;
    ...
};
```

The **Driver** doesn't do anything special, it just receives the notification from the **Device** that gpio interrupt has occurred, locates the appropriate registered **Component**'s callback object (based on the pin information provided by the **Device**), and uses **Event Loop** to schedule an execution of the **Component**'s callback together with information about input's value in event loop (non-interrupt) context.

Such generic GPIO **Driver** is already implemented in [embxx/driver/Gpio.h](https://github.com/arobenko/embxx/blob/master/embxx/driver/Gpio.h) file of [embxx](https://github.com/arobenko/embxx) library. The documentation can be found [here](https://dl.dropboxusercontent.com/u/46999418/embxx/driver_gpio_page.html).

### Button Component

The [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project has a simple button **Component**, implemented in [src/component/Button.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/component/Button.h). It configures provided GPIO line to be an input and to have both rising and falling edges interrupts. It also exposes simple interface to be able to monitor button presses and releases.
```cpp
template <typename TDriver,
          bool TActiveState,
          typename THandler = embxx::util::StaticFunction<void ()> >
class Button
{
public:
    typedef TDriver Driver;
    typedef typename Driver::PinIdType PinIdType;

    Button(Driver& driver, PinIdType pin);
    ~Button();

    bool isPressed() const;

    template <typename TFunc>
    void setPressedHandler(TFunc&& func);

    template <typename TFunc>
    void setReleasedHandler(TFunc&& func);
};
```

### Button Press Monitoring Application

The [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project also contains a simple application called [app_button](https://github.com/arobenko/embxx_on_rpi/tree/master/src/app/app_button). It monitors presses and releases of a single button connected to one of the GPIO lines. When the button is pressed, the led is turned on for 1 second and "Button Pressed" string is logged to UART. When the button is released, just "Button Released" string is logged to UART without influencing the led state. If new button press is recognised prior to 1 second timeout for the led being on, the led stays on and a new 1 second timer countdown is started.

Thanks to the [Device-Driver-Component](../basic_concepts/device_driver_component.md) model and all levels of abstractions, the application code is quite simple.
```cpp
int main() {
    auto& system = System::instance();

    // Configure uart
    auto& uart = system.uart();
    uart.configBaud(9600);
    uart.setWriteEnabled(true);

    // Allocate timer
    auto& timerMgr = system.timerMgr();
    auto timer = timerMgr.allocTimer();
    GASSERT(timer.isValid());

    // Set handlers for button press / release
    auto& button = system.button();
    button.setPressedHandler(
        std::bind(
            &buttonPressed,
            std::ref(timer)));

    button.setReleasedHandler(&buttonReleased);

    // Run event loop with enabled interrupts
    device::interrupt::enable();
    auto& el = system.eventLoop();
    el.run();

    GASSERT(0); // Mustn't exit
	return 0;
}
```

The code for "button pressed" is as following:
```cpp
void buttonPressed(System::TimerMgr::Timer& timer)
{
    static_cast<void>(timer);
    auto& system = System::instance();
    auto& el = system.eventLoop();
    auto& led = system.led();
    auto& log = system.log();

    SLOG(log, embxx::util::log::Info, "Button Pressed");

    timer.cancel();
    auto result = el.post(
        [&led]()
        {
            led.on();
        });
    GASSERT(result);
    static_cast<void>(result);

    static const auto WaitTime = std::chrono::seconds(1);
    timer.asyncWait(
        WaitTime,
        [&led](const embxx::error::ErrorStatus& es)
        {
            if (es == embxx::error::ErrorCode::Aborted) {
                return;
            }
            led.off();
        });
}
```

The code for "button release" is very simple:
```cpp
void buttonReleased()
{
    auto& system = System::instance();
    auto& log = system.log();

    SLOG(log, embxx::util::log::Info, "Button Released");
}
```


