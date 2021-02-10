## Event Loop

Most bare-metal embedded products require only two modes of operation:
* Interrupt (or service) mode
* Non-interrupt (or user) mode.

The job of the code, that is executed in interrupt mode, is to respond to hardware events (interrupts) by performing minimal job of updating various status registers and schedule proper handling of event (if applicable) to be executed in non-interrupt mode. In most projects the interrupt handlers are not prioritised, and the next hardware event (interrupt) won't be handled until the previously called interrupt handler returns, i.e. CPU is ready to return to non-interrupt mode. Therefore, it is important for the interrupt handler to do its job as quickly as possible.

There are multiple ways to schedule the execution of event handling code in non-interrupt mode from code being executed in interrupt mode. One of the easiest and straightforward ones is to have some kind of global flag that indicates that event has occurred and the processing is required:
```cpp
bool g_buttonPressed = false; 

void gpioInterruptHandler() 
{ 
    ... 
    if (/*button_gpio_recognised*/) { 
        g_buttonPressed = true; 
    } 
} 

int main(int argc, const char* argv[]) 
{ 
    ... 
    while (true) { // infinite event processing loop 
        enableInterrupts(); 
        ... 
        if (g_buttonPressed) { 
            disableInterrupt(); // avoid races 
            g_buttonPressed = false; 
            enableInterrupts(); 
            ... // Handle button press 
        } 
        ... 
        disableInterrupts(); 
        if (/* no_more_events */) {
            WFI(); // “Wait for interrupt” assembler instruction, 
                   // instruction will exit when there is pending 
                   // interrupt. 
        }
    } 
}
```

It is quite clear that this approach is not scalable, i.e. will quickly become a mess when number of hardware events the code needs to handle grows. The events may also be handled not in the same order they occurred, which may create undesired races and side effects on some systems.

Another widely used approach is to create a queue-like container (linked list or circular buffer) of event IDs which are handled in the similar event loop:
```cpp
enum EventId 
{ 
    EventId_ClockTick, 
    EventId_ButtonPress, 
    .... 
} 

Queue<EventId> events; 


void gpioInterruptHandler() 
{ 
    ... 
    if (/*button_gpio_recognised*/) { 
        events.push_back(EventId_ButtonPress); 
    } 
} 


int main(int argc, const char* argv[]) 
{ 
    ... 
    while (true) { // infinite event processing loop 
        enableInterrupts(); 
        ... 
        switch (events.front()) { 
        case EventId_ClockTick: 
            ... // handle clock tick 
            break; 
        
        case EventId_ButtonPress: 
            ... // handle button press 
            break; 
            ... 
        } 
        ... 
        disableInterrupts(); 
        events.pop_front(); // Remove processed event from queue 
       if (events.empty()) {
            WFI(); // “Wait for interrupt” assembler instruction, 
                   // instruction will exit when there is pending interrupt. 
        }
    } 
}
```

The approach above is a bit better, it processes events in the same order they occur, but still has its own disadvantages. Sometimes there is a need to attach some extra information for the processing of the event. Usually it is done using global variables, which introduces some extra complexity to the code and possibility for races. The handling of some events may have several internal stages and require busy wait(s) during the processing. These busy waits may significantly delay the processing of other pending events. The usual way to resolve this kind of problem is to create several state machines, that process this kind of events in stages. Most of Real-Time OSes provide an ability to create independent tasks (threads), that can be used to perform independent complex multiple staged workflows while the OS performs context switching between them. Still, the code can very quickly become too complex and difficult to maintain.

The approaches above are widely used in bare metal projects developed using C programming language. Using C++ language built-in features as well as ready to use classes from STL it is possible to simplify the complexity of the code and implement proper asynchronous handling of events, which is easier to debug and maintain. 

I would recommend using a queue of callable objects created by [std::bind()](http://en.cppreference.com/w/cpp/utility/functional/bind) expressions or [lambda functions](http://en.cppreference.com/w/cpp/language/lambda). The conventional C++ way would be using [std::list](http://en.cppreference.com/w/cpp/container/list) of [std::function](http://en.cppreference.com/w/cpp/utility/functional/function) objects. However, these classes use dynamic memory allocation and throw exceptions, which may be not suitable for every bare metal project. Anyway, let's just demonstrate the idea using these two classes:
```cpp
typedef std::list<std::function<void ()> > Queue; 
Queue handlers; 

template <typename TFunc> 
void addHandlerFromInterrupt(TFunc&& func) 
{ 
    // No need to disable interrupts.
    handlers.push_back(std::forward<TFunc>(func)); 
} 

template <typename TFunc> 
void addHandler(TFunc&& func) 
{ 
   // Protect against races with interrupt handlers 
    disableInterrupts(); 
    handlers.push_back(std::forward<TFunc>(func)); 
    enableInterrupts(); 
} 

void handleButtonPressStart() 
{ 
    ...// Start handling of button press event 
    handleButtonPressBusyWait(); 
} 

void handleButtonPressBusyWait() 
{ 
    if (/* some_condition */) { 
        handleButtonPressFinish(); 
        return; 
    } 
    
    // The condition is not true, need to wait, 
    // reschedule the execution of the same function.
    addHandler( 
        []() 
        { 
             handleButtonPressBusyWait(); 
        }); 
} 

void handleButtonPressFinish() 
{ 
    ...// Finalise handling of button press event. 
} 

void gpioInterruptHandler() 
{ 
    ... 
    if (/*button_gpio_recognised*/) { 
        addHandlerFromInterrupt( 
            []() 
            { 
		 // Will be executed in non-interrupt event loop.
                 handleButtonPressStart(); 
            }); 
    } 
} 

int main(int argc, const char* argv[]) 
{ 
    ... 
    while (true) { // infinite event processing loop 
        enableInterrupts(); 
        ... 
        auto& firstHandler = handlers.front(); 
        firstHandler(); // Execute scheduled callable object 
        ... 
        disableInterrupts(); 
        handlers.pop_front(); // Remove executed callable object 
                              // (function) from queue of handlers. 

        if (handlers.empty()) {
            WFI(); // “Wait for interrupt” assembler instruction, 
                   // instruction will exit when there is pending 
                   // interrupt. 
        }
    } 
}
```

This approach allows having complex processing of some events with many sub-stages and busy waits while still allowing other independent events being processed. All the handlers are executed in the same order they were pushed to the queue. There is an ability to bind multiples additional parameters together with the function call, which reduces a necessity to have global variables to pass values around. There is no need to maintain a list of various event IDs, explicitly define stages of state machine(s) or implement complex task switching between independent threads (tasks).

Now, let's try to get rid of dynamic memory allocation and possible exceptions. The only way to achieve this is to have a compile time constant that specifies the maximal size of the queue. The naive implementation would be using [StaticQueue](https://github.com/arobenko/embxx/blob/master/embxx/container/StaticQueue.h) of [StaticFunction](https://github.com/arobenko/embxx/blob/master/embxx/util/StaticFunction.h) objects described in [Basic Needs](../basic_needs/README.md) chapter. However, the [StaticFunction](https://github.com/arobenko/embxx/blob/master/embxx/util/StaticFunction.h) class definition requires compile time constant to specify the size of the area to store all the data of the callable object. It must be big enough to contain any possible callable object that will be pushed to the queue. For example:
```cpp
typedef embxx::util::StaticFunction<void (), sizeof(void*) * 10>  Func; 
typedef embxx::container::StaticQueue<Func, 1024> Queue;

Queue handlers;
…
handlers.push_back(std::bind(&func1, param1, param2)); // Will require size of only 3 values
...
handlers.push_back(
    std::bind(
        &func2, 
        param1, 
        param2,
        param3,
        param4)); // Will require size of only 5 values

handlers.push_back(
    std::bind(
        &func3, 
        param1, 
        param2,
        param3,
        param4,
        param5,
        param6,
        param7,
        param8,
        param9)); // Will consume the whole available space.
```

The queue will look like this:

![Queue of StaticFunction image](../image/event_loop_static_function.png)

It is quite clear that lots of space may be wasted and this approach must be optimised. What if we could push the callable object to the queue one after another regardless of their actual size with a bit of extra space overhead (such as pointer to v-table), that will help us to retrieve size of the object at runtime and remove appropriate number of bytes from such queue after the callable object did its job?

![Optimised queue image](../image/event_loop_optimised.png)

It looks much better. The space consumption is much more efficient.

To properly support this type of queue we must:
1. implement polymorphic behaviour when calling every handler with same interface.
1. implement polymorphic behaviour to retrieve the size of single handler in order to know how many bytes are to be removed from the queue after the handler has been called.
1. properly handle wrap-around cases when the pushed handler cannot fit into the area between the end of the queue and end of the allocated space.

The code of required classes will be like this:
```cpp
class Task 
{ 
public: 
    virtual ~Task() {} 

    virtual std::size_t getSize() const 
    { 
        return 1U; 
    } 

    virtual void exec() {} 
}; 

template <typename TTask> 
class TaskBound : public Task 
{ 
public: 

    // Size is minimal number of elements of size equal to sizeof(Task) 
    // that will be able to store this TaskBound object 
    static const std::size_t Size = 
        ((sizeof(TaskBound<typename std::decay<TTask>::type>) - 1) / 
                                                     sizeof(Task)) + 1; 

    explicit TaskBound(const TTask& task) 
      : task_(task) 
    { 
    } 

    explicit TaskBound(TTask&& task) 
      : task_(std::move(task)) 
    { 
    } 

    virtual ~TaskBound() {} 
    
    virtual std::size_t getSize() const 
    { 
        return Size; 
    } 

    virtual void exec() 
    { 
        task_(); 
    } 

private: 
    TTask task_; 
}; 
```

The definition of the Queue type will be:
```cpp
typedef typename 
    std::aligned_storage< 
        sizeof(Task), 
        std::alignment_of<Task>::value 
   >::type ArrayElemType; 

static const std::size_t ArraySize = TSize / sizeof(Task); 
typedef embxx::container::StaticQueue<ArrayElemType, ArraySize> Queue; 
```

`TSize` is a template parameter that specifies maximum size (in bytes) of the queue storage area.

The code of pushing new handler to the queue will look like this:
```cpp
template <typename TTask> 
bool addHandler(TTask&& task) 
{ 
    typedef TaskBound<typename std::decay<TTask>::type> TaskBoundType; 
    static_assert(
        std::alignment_of<Task>::value == std::alignment_of<TaskBoundType>::value, 
        "Alignment of TaskBound must be same as alignment of Task"); 

    static const std::size_t requiredQueueSize = TaskBoundType::Size; 

    auto placePtr = getAllocPlace(requiredQueueSize); 
    if (placePtr == nullptr) { 
        return false; 
    } 

    new (placePtr) TaskBoundType(std::forward<TTask>(task)); 
    return true; 
} 
```

Note, that job of `getAllocPlace()` function is to make sure that continuous storage area that is able to store the required callable object is created (by resizing the queue) and return pointer to this area.
```cpp
ArrayElemType* getAllocPlace(std::size_t requiredQueueSize) 
{ 
    auto invalidIter = queue_.invalidIter(); 
    while (true) 
    { 
        if ((queue_.capacity() - queue_.size()) < requiredQueueSize) { 
            return nullptr; 
        } 

        auto curSize = queue_.size(); 
        if (queue_.isLinearised()) { 
            auto dist = 
                static_cast<std::size_t>( 
                    std::distance(queue_.arrayTwo().second, invalidIter)); 
            if ((0 < dist) && (dist < requiredQueueSize)) { 
                queue_.resize(curSize + 1); 
                auto placePtr = static_cast<void*>(&queue_.back()); 
                new (placePtr) Task(); 
                continue; 
            } 
        } 
 
        queue_.resize(curSize + requiredQueueSize); 
        return &queue_[curSize]; 
    } 
} 
```

In case of wrap-around, when there is not enough space between the end of the queue and end of its storage area, number of simple `Task` objects which do nothing (the body of exec() function is empty) are pushed to fill the space till the end of storage area to make the queue non-linearised, which in turn will allow creation of continuous area of required size in the second half of the circular queue.

The event handling loop will be something like this:
```cpp
while (true) { 
    ... 
    // Get an access pointer to next handler 
    auto taskPtr = reinterpret_cast<Task*>(&queue_.front()); 
    auto sizeToRemove = taskPtr->getSize(); 

    // Execute the handler while allowing interrutps 
    enableInterrupts(); 
    taskPtr->exec(); 

    // Remove the handler information from the queue 
    taskPtr->~Task(); 
    disableInterrupts(); 
    queue_.popFront(sizeToRemove); 

    ... 
} 
```

The only remaining thing is to create a convenient and generic interface to be able to add new handlers for execution from both interrupt and non-interrupt contexts.

### Analogy with Threads

Before diving into implementation of such interface, I'd like to make an analogy between interrupt/non-interrupt execution modes and two threads. The inter-threads communication is managed using locks (such as [std::mutex](http://en.cppreference.com/w/cpp/thread/mutex)) and condition variables (such as [std::condition_variable_any](http://en.cppreference.com/w/cpp/thread/condition_variable_any)). Using this analogy the handlers execution loop (executed in non-interrupt thread) can be implemented like this:
```cpp
std::mutex lock_; 
std::condition_variable_any cond_; 
... 

while (true) { 
    lock_.lock(); 

    while (!queue_.isEmpty()) { 
        auto taskPtr = reinterpret_cast<Task*>(&queue_.front()); 
        auto sizeToRemove = taskPtr->getSize(); 
        lock_.unlock(); 

        // Executed with interrupts enabled 
        taskPtr->exec(); 
        taskPtr->~Task(); 

        lock_.lock(); 
        queue_.popFront(sizeToRemove); 
    } 

    // Still locked prior to wait 
    cond_.wait(lock_); 
    lock_.unlock(); 
} 
```

And adding new execution handler from any thread can be:
```cpp
template <typename TTask> 
bool addHandler(TTask&& task) 
{ 
   std::lock_guard<decltype(lock_)> guard(lock_); 
   ... // adding handler functionality 
   cond_.notify_all(); // notify the condition variable 
}
```

If we think about interrupt and non-interrupt execution modes as two threads, the locking in non-interrupt thread is equivalent to disabling interrupts; and waiting for condition variable to be notified is equivalent for waiting for interrupts (using `WFI` or `WFE` instructions in ARM architecture) while notification can be automatic due to pending interrupts or implemented using `SEV` instruction. However, our interrupt and non-interrupt mode threads  differ slightly from conventional threads. The non-interrupt mode one can be interrupted at any time by interrupt mode, while the interrupt mode “thread” won't be interrupted and doesn't actually need to protect itself from other thread's intervention.

The whole logic of event handling loop in non-interrupt context described above is generic except locking (disabling interrupts) and waiting for new handlers to be added (waiting for interrupts) which are platform and architecture specific. As I've mentioned before, the whole idea of using C++ instead of C in bare metal development is to be able to write and reuse generic code while providing minimal platform specific hardware control functionality. The [embxx](https://github.com/arobenko/embxx) library provides [EventLoop](https://github.com/arobenko/embxx/blob/master/embxx/util/EventLoop.h) class that receives the locking and condition variable classes as template parameters and manages safe addition of new handlers and in-order execution of the latter in non-interrupt context. 
```cpp
The class definition looks like this:
template <std::size_t TSize, typename Tlock, typename TCond> 
class EventLoop 
{ 
    ... 
};
```

The `TLock` class must expose the following public interface:
```cpp
class PlatformLock 
{ 
public: 
    // Locks out interrupt "thread". The function is called 
    // in non-interrupt context 
    void lock() {...} 

    // Restore previous state changed by "lock()" function, i.e. 
    // allow interrupts if they were disabled by lock(). 
    void unlock() {...} 

    // Same as lock(), but will be called when new handler is about to 
    // be added from interrupt handler. In normal case it should be an 
    // empty function, unless the interrupts are prioritised and there 
    // is a need to disable other interrupts from an interrupt handler 
    void lockInterruptCtx() {...} 

    // Same as unlock, but will be called in interrupt context. Should 
    // also be empty function when interrupts are not prioritised. 
    void unlockInterruptCtx() {...} 
};
```

The `TCond` class must expose the following public interface:
```cpp
class PlatformCond 
{ 
public: 
    // Receives the reference to lockable object that is locked 
    // (has lock() and unlock() member functions) and 
    // responsible to release the lock if needed and wait for 
    // notifications from other thread(s). After the notification 
    // occurs it must re-acquire the lock prior to returning. 
    template <typename TLock> 
    void wait(TLock& lock) {...} 

    // This function is used to notify condition that wait should 
    // be terminated. 
    void notify() {...} 
};
```

The example of such classes for Raspberry Pi platform may be found [here](https://github.com/arobenko/embxx_on_rpi/blob/master/src/device/EventLoopDevices.h).
```cpp
class InterruptLock 
{ 
public: 
    InterruptLock() 
        : flags_(0) {} 
 
    void lock() 
    { 
        __asm volatile("mrs %0, cpsr" : "=r" (flags_)); // store flags 
        __asm volatile("cpsid i"); // disable interrupts 
    } 

    void unlock() 
    { 
        if ((flags_ & IntMask) == 0) { 
            // Was previously enabled 
            __asm volatile("cpsie i"); // enable interrupts 
        } 
    } 

    void lockInterruptCtx() 
    { 
        // Nothing to do 
    } 

    void unlockInterruptCtx() 
    { 
        // Nothing to do 
    } 

private: 
    volatile std::uint32_t flags_; 
    static const std::uint32_t IntMask = 1U << 7; 
}; 

class WaitCond 
{ 
public: 
    template <typename TLock> 
    void wait(TLock& lock) 
    { 
        // no need to unlock (re-enable interrupts) 
        static_cast<void>(lock); 
        __asm volatile("wfi"); 
    } 

    void notify() 
    { 
        // Nothing to do, pending interrupt will cause wfi
        // to exit even with interrupts disabled 
    } 
}; 
```

The [EventLoop](https://github.com/arobenko/embxx/blob/master/embxx/util/EventLoop.h) class exposes the following public interface:
```cpp
template <std::size_t TSize, typename Tlock, typename TCond> 
class EventLoop 
{ 
public: 
    ... 
    /// @brief Post new handler for execution. 
    /// @details Acquires regular context lock. The task is added to 
    ///          the execution queue. If the execution queue is empty 
    ///          before the new handler is added, the condition 
    ///          variable is signalled by calling its notify() member 
    ///          function. 
    /// @param[in] task R-value reference to new handler functor. 
    /// @return true in case the handler was successfully posted, 
    ///         false if there is not enough space in the execution 
    ///         queue. 
    template <typename TTask> 
    bool post(TTask&& task); 

    /// @brief Post new handler for execution from interrupt context. 
    /// @details Acquires interrupt context lock. The task is added to 
    ///          the execution queue. If the execution queue is empty 
    ///          before the new handler is added, the condition variable
    ///          is signalled by calling its notify() member function. 
    /// @param[in] task R-value reference to new handler functor. 
    /// @return true in case the handler was successfully posted, false 
    ///         if there is not enough space in the execution queue. 
    template <typename TTask> 
    bool postInterruptCtx(TTask&& task); 

    /// @brief Event loop execution function. 
    /// @details The function keeps executing posted handlers until 
    ///          none are left. When execution queue becomes empty the 
    ///          wait(...) member function of the condition variable 
    ///          gets called to execute blocking wait for new handlers. 
    ///          When new handler is added, the condition variable will 
    ///          be signalled and blocking wait is expected to be 
    ///          terminated to continue execution of the event loop. 
    ///          This function never exits unless stop() was called to
    ///          terminate the execution. After stopping the main
    ///          loop, use reset() member function to enable the loop 
    ///          to be executed again.
    void run(); 

    /// @brief Stop execution of the event loop. 
    /// @details The execution may not be stopped immediately. If there
    ///          is an event handler being executed, the loop will be 
    ///          stopped after the execution of the handler is finished. 
    void stop(); 

    /// @brief Reset the state of the event loop. 
    /// @details Clear the queue of registered event handlers and 
    ///          resets the "stopped" flag to allow new event loop 
    ///          execution. 
    void reset(); 
}: 
```

I'll leave the implementation of the functions above as an exercise to the reader. Don't forget to call `notify()` member function of condition variable when adding new handler to the empty queue. 

If needed, the reference implementation can be found [here](https://github.com/arobenko/embxx/blob/master/embxx/util/EventLoop.h).

### Busy Loops 

The event loop described above is an easy and convenient way to implement soft real-time systems. However, the main rule with such architecture is: **DON'T DO BUSY LOOPS!** It means, if there is a real need to perform a busy wait before proceeding to the next stage, do it by letting other events being handled as well. The `EventLoop` class also provides `busyWait()` member function that does exactly that. 
```cpp
template <std::size_t TSize, typename Tlock, typename TCond> 
class EventLoop 
{ 
public: 
    ... 
    /// @brief Perform busy wait. 
    /// @details Executes busy wait while allowing other event handlers 
    ///          posted by interrupt handlers being processed. 
    /// @tparam TPred Predicate class type, must define 
    ///         @code bool operator()(); @endcode 
    ///         that return true in case busy wait must be terminated. 
    /// @tparam TFunc Functor class that will be executed when wait is 
    ///         complete. It must define 
    ///         @code void operator()(); @endcode 
    /// @param pred Any type of reference to predicate object 
    /// @param func Any type of reference to "wait complete" function. 
    /// @pre The event loop must have enough space to repost the call 
    ///      to busyWait(). Note that there is no wait to notify the 
    ///      caller if post operation fails. In debug compilation mode
    ///      there will be an assertion failure in case call to post()
    ///      returned false, in release compilation mode the failure 
    ///      will be silent. 
    template <typename TPred, typename TFunc> 
    void busyWait(TPred&& pred, TFunc&& func) 
    { 
        if (pred()) { 
            bool result = post(std::forward<TFunc>(func)); 
            GASSERT(result); 
            static_cast<void>(result); 
            return; 
        } 

        bool result = post( 
            [this, pred, func]() 
            { 
                busyWait(std::move(pred), std::move(func)); 
            }); 
        GASSERT(result); 
        static_cast<void>(result); 
    } 
};
```

