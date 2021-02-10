# Timer

It is customary in bare metal development to flash leds in the first application (instead of writing "Hello world"). However most tutorials show how to do it synchronously using loops to wait some time before changing state of the led. I'm going to describe how to do it asynchronously using timer interrupt in conjunction with [Event Loop](../basic_concepts/event_loop.md).

Almost every embedded platform has usually one or two timer peripherals. One such peripheral can be programmed to provide an interrupt after some period of time. However, there may be a need to have multiple timers that can be activated independently at the same time. It is quite clear that there should be an entity that receives all the wait requests from various **Component**s in non-interrupt context, then queues the wait requests internally,  programs the timer peripheral to provide an interrupt after some time, and finally reports the completion to appropriate **Component** via callback also in non-interrupt (event loop) context.

Such entity can be a generic (platform independent) **Driver**, if it is provided with platform specific **Device** object, that exposes some predefined public interface and controls the actual platform specific hardware.

![Image: Timer Manager](../image/timer_mgr.png)

The asynchronous timer event handling follows the same pattern described in [Device-Driver-Component](../basic_concepts/device_driver_component.md) chapter.

### Assigning Wait Complete Callback

Just like described in [Device-Driver-Component](../basic_concepts/device_driver_component.md) chapter the **Driver** needs to provide the "Wait Complete" callback object to be called when timer interrupt occurs. The assignment is usually performed during initialisation/construction stage of the **Driver**:

![Image: Assigning callback](../image/timer_wait_callback.png)

### Starting Asynchronous Wait

![Image: Starting Asynchronous Wait](../image/timer_wait_start1.png)

The **Driver** must be able to support multiple wait requests from various **Components** and manage the internal queue accordingly. In the chart above the timer peripheral activated on the first `asyncWait()` request. When the second request is issued (assuming `timeout1 < timeout2` and existing wait mustn't be stopped), the **Driver** must prevent the completion of the currently scheduled timer countdown being reported in interrupt context while interfering with an update to internal data structures. The interrupts are disabled by calling `suspendWait()` member function of the **Device**. The call to the `suspendWait()` returns `true`, which means the interrupts are successfully disabled and it is safe to update internal data structures. If the call to `suspendWait()` returns `false`, it means that the interrupt has already occurred and there is no existing wait in progress, i.e. the second `asyncWait()` actually becomes a first one in the new sequence.

There also may be a case when `timeout2 < timeout1` which means the order of the timeout requests must be re-evaluated, and new wait re-programmed.

![Image: Starting Asynchronous Wait](../image/timer_wait_start2.png)

The **Driver** must be able to cancel the existing timer countdown, evaluate how much time has passed since the first request, evaluate the new values to reprogram the timer **Device** countdown again.

### Completing Asynchronous Wait

![Image: Completing Asynchronous Wait](../image/timer_wait_complete.png)

Due to the fact that **Driver** may receive multiple independent wait requests, it must reprogram the next wait (if such exists) while running in interrupt mode. Please pay attention to `InterruptCtx()` tag parameter passed to the `startWait()` member function of the **Device**. It indicates that the request is executed in interrupt context, while the same request used `EventLoopCtx()` as the tag parameter to specify that the call was performed in event loop (non-interrupt) context.

### Canceling Asynchronous Wait

If there is a request to cancel the currently executed wait, the **Driver** must receive the information about the elapsed time and reprogram the next wait if such exists.

![Image: Canceling Asynchronous Operation](../image/timer_wait_cancel1.png)

If the cancellation request to some other wait, that hasn't been forwarded to the **Device**, the **Driver** just needs to update its internal data structures without canceling currently performed timer countdown.

![Image: Canceling Asynchronous Operation](../image/timer_wait_cancel2.png)

The unsuccessful attempts to cancel wait is performed in exactly the same way as described in [Device-Driver-Component](../basic_concepts/device_driver_component.md) chapter.

![Image: Canceling Asynchronous Operation](../image/timer_wait_cancel3.png)

### Identifying Wait Requests

There is obviously a need to have some kind of identification of the wait requests in order to be able to cancel some specific request while keeping the rest in waiting queue. One approach would be to have some kind of a handle which can be used during the cancellation request:
```cpp
class MyTimerDriver
{
public:
    typedef ... Handle;

    Handle asyncWait(...);
    
    void cancelWait(Handle handle);
};
```

Another one is to hide the handle in some wrapper class, which makes it a bit safer to use:
```cpp
class MyTimerDriver
{
public:
    
    typedef ... Handle;

    class Timer
    {
    public:
        Timer(MyTimerDriver& mgr, Handle handle)
          : mgr_(mgr),
            handle_(handle)
        {
        }

        ~Timer()
        {
            ... // Invalidate the allocated handle
        }

        void asyncWait(...)
        {
            mgr_.asyncWait(handle_, ...)
        }
        
        void cancelWait()
        {
            mgr_.cancelWait(handle_);
        }

    private:
	MyTimerDriver& mgr_;
        Handle handle_;
    };

    Timer allocTimer()
    {
        auto someHandle = ...;
        return Timer(*this, someHandle)
    }

private:

    friend class TimerMgr::Timer;

    void asyncWait(Handle handle, ...);
    
    void cancelWait(Handle handle);
};
```

The **Driver** itself has only one public function `allocTimer()`. It is used to allocate the `Timer` object. All the wait and/or cancel requests are issued to this timer object directly, which is declared to be a `friend` of the **Driver** class, i.e. it is able to call private functions of the latter using the handle it has. The destructor of the `Timer` makes sure that the handle is properly invalidated.

```cpp
MyTimerDriver driver(...); 
auto timer = driver.allocTimer();
timer.asyncWait(...);
...
timer.cancelWait();
...
```

The second approach is a bit safer than the first one and it is used in the implementation of such generic "Timer Management Driver" in [embxx](https://github.com/arobenko/embxx) library.

### Specifying the Wait Duration

The timer **Device** is platform specific. Some platforms may support wait duration granularity of a microsecond, others can achieve only a millisecond. It usually depends on the system clock speed. However, when using generic **Driver** and/or **Component** there is a need to be able to write platform independent code that performs wait of the specified duration regardless of the **Device** in use. The **S**tandard **T**emplate **L**ibrary (**STL**) of C++11 standard provides convenient [Date and Time Utilities](http://en.cppreference.com/w/cpp/chrono) that make such usage possible.

In case the **Device** declares a minimal wait duration unit using [std::chrono::duration](http://en.cppreference.com/w/cpp/chrono/duration) type, the **Driver** may use [std::chrono::duration_cast](http://en.cppreference.com/w/cpp/chrono/duration/duration_cast) to convert the requested wait duration to supported duration units.
```cpp
class MyTimerDevice
{
public:
    typedef std::chrono::duration<unsigned, std::milli> 
                                                WaitTimeUnitDuration;

    typedef embxx::device::context::EventLoop EventLoopCtx;

    void startWait(WaitTimeUnitDuration::rep count, EventLoopCtx) {...}
    ...
};
```

In the example above the minimal supported duration unit (`WaitTimeUnitDuration`) is declared to be 1 millisecond. Please note that `startWait()` member function expects to receive number of wait units, i.e. milliseconds as its first parameter.

Then the definition of the `asyncWait()` member function of the **Driver** may be defined like this:
```cpp
template <typename TDevice, ...>
class MyTimerDriver
{
public:
    typedef typename TDevice::WaitTimeUnitDuration WaitTimeUnitDuration
    class Timer
    {
    public:
        template <typename TRep, typename TPeriod, typename TFunc>
        void asyncWait(
            const std::chrono::duration<TRep, TPeriod>& waitTime,
            TFunc&& func)
        {
            auto castedWaitDuration =
                std::chrono::duration_cast<WaitTimeUnitDuration>(waitTime);
            auto waitUnits = castedWaitDuration.count();
            ... // Call the asyncWait() of the driver with waitUnits as
                // first parameter.
        }

    };
};
```

In the example above the call below will perform correct adjustment of the duration and will measure the same timeout with any **Device** whether the latter expects milliseconds or microseconds in its `startWait()` member function.
```cpp
timer.asyncWait(std::chrono::seconds(5), ...);
```

In case the developer tries to execute a wait of several microseconds when **Driver** supports only milliseconds granularity, the compilation will fail.
```cpp
timer.asyncWait(std::chrono::microseconds(5), ...);
```

### Driver Implementation

The timer management **Driver** is a generic layer. It must work on any platform with any timer **Device** object that exposes the right interface.

Such **Driver** is already implemented in [embxx](https://github.com/arobenko/embxx) library as `embxx::driver::TimerMgr` and resides in [embxx/driver/TimerMgr.h](https://github.com/arobenko/embxx/blob/master/embxx/driver/TimerMgr.h) while platform specific (Raspberry Pi) peripheral control object is implemented in [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project as `device::Timer` and resides in [src/device/Timer.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/Timer.h).

The detailed documentation for `embxx::driver::TimerMgr` can be found [here](https://dl.dropboxusercontent.com/u/46999418/embxx/driver_timer_mgr_page.html).

The `embxx::driver::TimerMgr` is defined like this:
```cpp
template <typename TDevice,
          typename TEventLoop,
          std::size_t TMaxTimers,
          typename TTimeoutHandler = embxx::util::StaticFunction<void (const embxx::error::ErrorStatus&)> >
class TimerMgr
{
public:
    TimerMgr(TDevice& device, TEventLoop& el);
      : device_(device),
        el_(el)
    {
        ...
    }

    ...

private:
    struct TimerInfo {
        TTimeoutHandler handler_; //
        ...;                      // Some other internal data
    }

    // Internal data structures to track all the scheduled
    // wait requests.
    std::array<TimerInfo, TMaxTimers> infos_;

    TDevice& device_;
    TEventLoop& el_;
    ...
};

```

The `TDevice` template parameter is Platform specific control class for timer peripheral.

The `TEventLoop` template parameter is the class of the [Event Loop](../basic_concepts/event_loop.md).

The `TMaxTimers` template parameters specifies the maximal number of timer objects the `TimerMgr` will be able to allocate. This parameter is required because `embxx::driver::TimerMgr` was designed to be used in the systems without dynamic memory allocation. If dynamic memory allocation is allowed, then it is quite easy to implement similar functionality without this limitation.

The `TTimeoutHandler` template parameter specifies type of the timeout callback object. This object must have `void (const embxx::error::ErrorStatus&)` signature and expose similar interface to [std::function](http://en.cppreference.com/w/cpp/utility/functional/function) or [embxx::util::StaticFunction](https://dl.dropboxusercontent.com/u/46999418/embxx/classembxx_1_1util_1_1StaticFunction_3_01TRet_07TArgs_8_8_8_08_00_01TSize_01_4.html).

The `embxx::driver::TimerMgr` exposes the following public interface:
```cpp
template <...>
class TimerMgr
{
public:
    class Timer 
    {
    public:
        // Destructor, removes Timer record from internal
        // data structures of TimerMgr
        ~Timer() {...}

        // Activates asyncrhonous wait
        void asyncWait(...) {...}

        // Cancels scheduled asynchronous wait
        void cancel() {...}
    };

    // Allocate timer object
    Timer allocTimer() {...}
  
private:
    // Allows usage of non-exposed private functions of
    // TimerMgr
    friend class TimerMgr::Timer;
    ...
};
```

The reader may notice that `embxx::driver::TimerMgr` exposes only one public function: `Timer allocTimer();`. This function returns simple `TimerMgr::Timer` object which can be used to schedule new wait as well as cancel the previous wait request. 
Also note that `TimerMgr::Timer` class is declared to be a `friend` of `TimerMgr`. This is required to allow seamless delegation of the wait/cancel request from `TimerMgr::Timer` to `TimerMgr` which is responsible for managing multiple simultaneous wait requests and delegating them one by one to the the actual hardware control object.

Then the led flashing application (implemented in [src/app/app_led_flash](https://github.com/arobenko/embxx_on_rpi/tree/master/src/app/app_led_flash)) can be as simple as the code below:
```cpp
namespace
{

const auto LedChangeStateTimeout = std::chrono::milliseconds(500);

template <typename TTimer>
void ledOff(
    TTimer& timer,
    System::Led& led);

template <typename TTimer>
void ledOn(
    TTimer& timer,
    System::Led& led)
{
    led.on();

    timer.asyncWait(
        LedChangeStateTimeout,
        [&timer, &led](const embxx::error::ErrorStatus& status)
        {
            static_cast<void>(status);
            ledOff(timer, led);
        });
}

template <typename TTimer>
void ledOff(
    TTimer& timer,
    System::Led& led)
{
    led.off();

    timer.asyncWait(
        std::chrono::milliseconds(LedChangeStateTimeout),
        [&timer, &led](const embxx::error::ErrorStatus& status)
        {
            static_cast<void>(status);
            ledOn(timer, led);
        });
}

}  // namespace

int main() {
    // Get reference to TimerMgr object
    auto& system = System::instance();
    auto& timerMgr = system.timerMgr();

    // Allocate timer
    auto timer = timerMgr.allocTimer();

    // Start flashing with initial state to be OFF
    device::interrupt::enable();
    ledOff(timer, led);

    // Run the event loop
    auto& el = system.eventLoop();
    el.run();

    GASSERT(0); // Mustn't exit
    return 0;
}
```

## Platform Specific Timer Device

As it was already mentioned earlier, the `embxx::driver::TimerMgr` is a generic **Driver** class that does most of the work of managing and scheduling independent wait requests. It requires support from low level timer **Device** object to program the actual hardware of the platform the code runs on. The `embxx::driver::TimerMgr` is defined to receive the **Device** class as template parameter as well as reference to the **Device** timer object in the constructor. The **Driver** doesn't know the exact **Device** type, but expects it to expose certain public interface:
```cpp
template <typename TDevice, typename TEventLoop, ...>
class TimerMgr
{
public:
    TimerMgr(TDevice& device, TEventLoop& el);
    ...
};

```

The timer control **Device** class must expose the following public interface:
1. Define `WaitTimeUnitDuration` type as variation of [std::chrono::duration](http://en.cppreference.com/w/cpp/chrono/duration/duration_cast) that specifies duration of single wait unit supported by the **Device**.
    ```cpp
    typedef std::chrono::duration<...> WaitTimeUnitDuration;
    ```
1. Function to set the callback object to be invoked from timer interrupt:
    ```cpp
    template <typename TFunc>
    void setWaitCompleteCallback(TFunc&& func);

    ```
1. Functions to start timer countdown in both event loop (non-interrupt) and interrupt contexts:
    ```cpp
    void startWait(
        WaitTimeUnitDuration::rep waitTime, // num of wait units
        embxx::device::context::EventLoop context);
    void startWait(
        WaitTimeUnitDuration::rep waitTime, // num of wait units
        embxx::device::context::Interrupt context);
    ```
1. Function to cancel timer countdown in event loop (non-interrupt) context. The function must return true in case the wait was actually canceled and false when there is no wait in progress.
    ```cpp
    bool cancelWait(embxx::device::context::EventLoop context);
    ```
1. Function to suspend countdown (disable interrupts while the actual wait countdown is not stopped) in event loop (non-interrupt) context. The function must return true in case the wait was actually suspended and false when there is no wait in progress. The call to this function will be followed either by `resumeWait()` or by `cancelWait()`.
    ```cpp
    bool suspendWait(embxx::device::context::EventLoop context);
    ```
1. Function to resume countdown in event loop (non-interrupt) context.
    ```cpp
    void resumeWait(embxx::device::context::EventLoop context);
    ```
1. Function to retrieve elapsed time of the last executed wait. It will be called right after the `cancelWait()`.
    ```cpp
    WaitTimeUnitDuration::rep getElapsed(embxx::device::context::EventLoop context) const;
    ```

The definition and implementation of such timer device for Raspberry Pi platform can be found in [src/device/Timer.h](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/Timer.h) file of [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project.

