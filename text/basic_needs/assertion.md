## Assertion

One of the basic needs during the development is having an ability to test various assumptions and invariants in runtime when compiling the application in DEBUG mode and remove the checks when compiling the application in RELEASE mode. The standard C++ reuses `assert()` macro from standard C library. 
```cpp
#include <cassert>
…
assert(some_condition);
```

The `assert()` macro evaluates to nothing in case `NDEBUG` symbol is defined, otherwise it evaluates the condition. If the condition doesn't return `true`, it calls the `__assert_fail` function, provided by standard library, which in turn calls `printf` to print error message to standard output followed by the call to `abort` function, which is supposed to terminate an application.

Both `printf` and `abort` functions are provided by standard library. However, `printf` will require the implementation of `_write` function to print characters to the debug output terminal, and `abort` will require implementation of `_exit` function to terminate the application.

If standard library is excluded from the compilation (using `-nostdlib` compilation option), the compilation will fail with “undefined reference to `__assert_func`" error message. The developer will have to implement this function with correct signature. To retrieve the correct signature you will have to open `assert.h` standard header provided by your compiler. It will be something like this:
```cpp
void __assert_fail (const char *expr, const char *file, unsigned int line, const char *function) __attribute__ ((__noreturn__)); 
```

The attribute specifies that this function doesn't return, so the compiler will generate a call to it without setting any address to return to.

The conclusion from all the stated above is that using standard `assert()` macro is possible, but somewhat inflexible. It is possible to access only global variables from the functions described above, i.e. if there is a need to flash a led to indicate assertion failure, then its control must be accessible through global variables, which is a bit ugly. Another disadvantage of this approach is that there are no convenient means to change the behaviour of the assert failure functionality and after a while restore the original behaviour. Such behaviour may be helpful to better identify the location of the assert that has failed. For example, override the default assert failure behaviour with activating a specific led at the entrance of some function, and restore the original assertion failure behaviour when function returns.

Below is a short description of a better way to handle assert checks and failures. The code is in [embxx](https://github.com/arobenko/embxx) library and can be reviewed [here](https://github.com/arobenko/embxx/blob/master/embxx/util/Assert.h).

To resolve the problems described above and to handle the assertions C++ way we will have to create generic assertion failure handling abstract class:
```cpp
class Assert 
{ 
public: 
    virtual void fail( 
        const char* expr, 
        const char* file, 
        unsigned int line, 
        const char* function) = 0; 
}; 
```

When implementing custom project specific assertion failure behaviour inherit from the class above:
```cpp
#include "embxx/util/Assert.h" 

typedef ... Led; 
class LedOnAssert : public embxx::util::Assert 
{ 
public: 

    LedOnAssert(Led& led) 
        : led_(led) 
    { 
    } 

    virtual void fail( 
        const char* expr, 
        const char* file, 
        unsigned int line, 
        const char* function) 
    { 
        led_.on(); 
        while (true) {;} 
    } 

private: 
    Led& led_; 
}; 
```

To manage an object of the class above, we will have to create a singleton class with static instance. It will store a pointer to the currently registered assertion failure behaviour:
```cpp
class AssertManager 
{ 
public: 
   static AssertManager& instance() 
    { 
        static AssertManager mgr; 
        return mgr; 
    } 

    Assert* reset(Assert* newAssert = nullptr) 
    { 
        auto prevAssert = assert_; 
        assert_ = newAssert; 
        return prevAssert; 
    } 

    Assert* getAssert() 
    { 
        return assert_; 
    } 

    bool hasAssertRegistered() const 
    { 
        return assert_ != nullptr; 
    } 

    void infiniteLoop() 
    { 
        while (true) {}; 
    } 

private: 
    AssertManager() : assert_(nullptr) {} 

    Assert* assert_; 
}; 
```

The `reset` member function registers new object that manages assertion failure behaviour and returns previous one, which can be used later to restore original behaviour.

We will require a new macro to check assertion condition and invoke registered failing behaviour:
```cpp
#ifndef NDEBUG 

#define GASSERT(expr) \ 
    ((expr)                               \ 
      ? static_cast<void>(0)                     \ 
      : (embxx::util::AssertManager::instance().hasAssertRegistered() \ 
            ? embxx::util::AssertManager::instance().getAssert()->fail( \ 
                #expr, __FILE__, __LINE__, GASSERT_FUNCTION_STR) \ 
            : embxx::util::AssertManager::instance().infiniteLoop())) 

#else // #ifndef NDEBUG 

#define GASSERT(expr) static_cast<void>(0) 
 
#endif // #ifndef NDEBUG 
```

Then in case of condition check failure, the `GASSERT()` macro checks whether any custom assertion failure functionality registered and invokes its virtual `fail` function. If not, then infinite loop is executed.

To complete the whole picture we have to provide a convenient way to register new assertion failure behaviours: 
```cpp
template < typename TAssert> 
class EnableAssert 
{ 
    static_assert(std::is_base_of<Assert, TAssert>::value, 
        "TAssert class must be derived class of Assert"); 
public: 
    typedef TAssert AssertType; 

    template<typename... Params> 
    EnableAssert(Params&&... args) 
        : assert_(std::forward<Params>(args)...), 
          prevAssert_(AssertManager::instance().reset(&assert_))
    { 
    } 

    ~EnableAssert() 
    { 
        AssertManager::instance().reset(prevAssert_); 
    } 

private: 
    AssertType assert_; 
    Assert* prevAssert_; 
}; 
```

From now on, all we have do is to instantiate object of `EnableAssert` with the behaviour that we want. Note that constructor of `EnableAssert` class can receive any number of parameters and forwards them to the constructor of the internal `assert_` object.
```cpp
int main (int argc, const char* argv[]) 
{ 
    ... 
    Led led; 
    embxx::util::EnableAssert<LedOnAssert> assertion(led); 

    ... // Rest of the code
} 
```

If there is a need to temporarily override the previous assertion failure behaviour, just create another `EnableAssert` object. Once the latter is out of scope (the object is destructed), previous behaviour will be restored.
```cpp
int main (int argc, const char* argv[]) 
{ 
    ... 
    Led led; 
    embxx::util::EnableAssert<LedOnAssert> assertion(led); 

    ... 
    { 
        embxx::util::EnableAssert<OtherAssert> otherAssertion(.../* some params */); 
        ... 
    }  // restore previous registered behaviour – LedOnAssert.
} 
```

**SUMMARY**: The approach described above provides a flexible and convenient way to control how the failures of various debug mode checks are reported to the developer. All the modules in [embxx](https://github.com/arobenko/embxx) library use the `GASSERT()` macro to verify their pre- and post-conditions as well as internal assumptions.

Extra documentation for the Generic Assert functionality can be found [here](https://dl.dropboxusercontent.com/u/46999418/embxx/util_assert_page.html).


