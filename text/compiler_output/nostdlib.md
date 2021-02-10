## Removing Standard Library and C++ Runtime

Due to platform RAM/ROM limitations it may be required to exclude not just support for exceptions and RTTI (compiling with `-fno-exceptions` `-fno-unwind-tables` `-fno-rtti`), but for dynamic memory allocation too. The latter includes passing `-nostdlib` option to the compiler.
In case when standard library is excluded, there is no startup code help 
provided by the compiler, the developer will have to implement all the startup stages: 
* updating the interrupt vector table
* setting up correct stack pointers for all the modes of execution 
* zeroing `.bss` section 
* calling initialisation functions for global objects
* calling “main” function.

[Here](https://github.com/arobenko/embxx_on_rpi/blob/master/src/asm/startup.s) is an example of such startup code.

There also may be a need to provide an implementation of some functions or 
definition of some global symbols. For example, if 
[std::copy](http://en.cppreference.com/w/cpp/algorithm/copy) algorithm is used 
to copy multiple objects from place to place, the compiler might decide to use 
[memcpy](http://en.cppreference.com/w/c/string/byte/memcpy) function provided by 
the standard library, and as the result the build process will fail with 
“undefined reference” error. The same way, usage of 
[std::fill](http://en.cppreference.com/w/cpp/algorithm/fill) algorithm may 
require [memset](http://en.cppreference.com/w/c/string/byte/memset) function. 
Be ready to implement them when needed.

Another example is having call to 
[std::bind](http://en.cppreference.com/w/cpp/utility/functional/bind) function 
with [std::placeholders::_1](http://en.cppreference.com/w/cpp/utility/functional/placeholders), [std::placeholders::_2](http://en.cppreference.com/w/cpp/utility/functional/placeholders), etc. 
There will be a need to define these placeholders as global symbols:
```cpp
#include <functional>
namespace std 
{ 
namespace placeholders 
{ 

decltype(std::placeholders::_1) _1; 
decltype(std::placeholders::_2) _2; 
decltype(std::placeholders::_3) _3; 
decltype(std::placeholders::_4) _4; 

}  // namespace placeholders 
}  // namespace std 
```

Even if there is a need for the standard library in the product being developed, 
it may be a good exercise as well as good debugging technique to temporarily 
exclude it from the compilation. The compilation will probably fail in the 
linking stage. The list of missing symbols and/or functions will provide a good 
indication of what missing functionality is provided by the library. The 
developer may notice that some components still require exceptions handling, 
for example, resulting int the binary image being too big.

