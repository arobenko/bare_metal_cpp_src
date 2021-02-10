## Dynamic Memory Allocation
Let's try to compile simple application that uses dynamic memory allocation. The [test_cpp_vector](https://github.com/arobenko/embxx_on_rpi/tree/master/src/test_cpp/test_cpp_vector) application contains the following code:

```cpp
std::vector<int> v; 
static const int MaxVecSize = 256; 
for (int i = 0; i < MaxVecSize; ++i) { 
    v.push_back(i); 
}
```

It may happen that linking operation will fail with multiple referenced symbols being undefined:
```
unwind-arm.c:(.text+0x224): undefined reference to `__exidx_end' 
unwind-arm.c:(.text+0x228): undefined reference to `__exidx_start' 
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-abort.o): In function `abort': 
abort.c:(.text.abort+0x10): undefined reference to `_exit' 
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-sbrkr.o): In function `_sbrk_r': 
sbrkr.c:(.text._sbrk_r+0x18): undefined reference to `_sbrk' 
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-signalr.o): In function `_kill_r': 
signalr.c:(.text._kill_r+0x1c): undefined reference to `_kill' 
/usr/bin/../lib/gcc/arm-none-eabi/4.8.3/../../../../arm-none-eabi/lib/libc.a(lib_a-signalr.o): In function `_getpid_r': 
signalr.c:(.text._getpid_r+0x4): undefined reference to `_getpid' 
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

The symbols `__exidx_start` and `__exidx_end` are required to indicate start and end of `.ARM.exidx` section. It is used for exception handling. They must be defined in the linker script:
```
.ARM.exidx : 
{ 
    __exidx_start = .; 
    *(.ARM.exidx* .gnu.linkonce.armexidx.*) 
    __exidx_end = .; 
} >RAM 
```

The dynamic memory allocation will require implementation of `_sbrk` function which will be used to allocate chunks of memory for the C/C++ heap management.

All other symbols will be required to properly support exceptions which are used 
by C++ heap management system. 
[Here](https://sourceware.org/newlib/libc.html#Syscalls) is a good resource, 
that lists all the system calls, the developer may need to implement, to get 
the application compiled.

Now, after successful compilation, take a good look at the size of the images of two sample applications we compiled. The paths are `<build_dir>/src/test_cpp/test_cpp_simple/kernel.img` and `<build_dir>/src/test_cpp/test_cpp_vector/kernel.img`.

**Side note**: The image can be generated out of elf binary using the following instruction:
    > arm-none-eabi-objcopy <elf_executable> -O binary <binary_image_path>

You may notice that size of [test_cpp_vector](https://github.com/arobenko/embxx_on_rpi/tree/master/src/test_cpp/test_cpp_vector) image is greater by approximately 100K than [test_cpp_simple](https://github.com/arobenko/embxx_on_rpi/tree/master/src/test_cpp/test_cpp_simple). It is due to C++ heap management and exceptions handling. Let's try to see what happens to the size of the application if "C++" heap is replaced with “C” one without exceptions. You will have to override all the global C++ operators responsible for memory allocation/deallocation:
```cpp
#include <cstdlib> 
#include <new> 

void* operator new(size_t size) noexcept 
{ 
    return malloc(size); 
} 

void operator delete(void *p) noexcept 
{ 
    free(p); 
} 

void* operator new[](size_t size) noexcept 
{ 
    return operator new(size); // Same as regular new
} 

void operator delete[](void *p) noexcept 
{ 
    operator delete(p); // Same as regular delete
} 

void* operator new(size_t size, std::nothrow_t) noexcept 
{ 
    return operator new(size); // Same as regular new 
} 

void operator delete(void *p,  std::nothrow_t) noexcept 
{ 
    operator delete(p); // Same as regular delete
} 

void* operator new[](size_t size, std::nothrow_t) noexcept 
{ 
    return operator new(size); // Same as regular new
} 
 
void operator delete[](void *p,  std::nothrow_t) noexcept 
{ 
    operator delete(p); // Same as regular delete
} 
```

Please compile the [test_cpp_vector](https://github.com/arobenko/embxx_on_rpi/tree/master/src/test_cpp/test_cpp_vector) application again, create its image and take a look at its size. It will be much closer to the size of the [test_cpp_simple](https://github.com/arobenko/embxx_on_rpi/tree/master/src/test_cpp/test_cpp_simple) image. In fact, you may not even need majority of the system call functions you have implemented before. Try to remove them one by one and see whether linker still reports “undefined reference” to these symbols. 

**CONCLUSION**: Usage of C++ heap brings a significant code size overhead. 
It is a good practice to override implementation of `new` and `delete` operators 
with usage of `malloc` and `free` when using C++ in bare metal development. 
Note that in this case, if memory allocation fails 
[nullptr](http://en.cppreference.com/w/cpp/types/nullptr_t) will be returned 
instead of throwing 
[std::bad_alloc](http://en.cppreference.com/w/cpp/memory/new/bad_alloc) exception, 
so beware of third party C++ libraries that count on exception been thrown and 
do not check the returned value form 
[operator new](http://en.cppreference.com/w/cpp/memory/new/operator_new).

### Excluding Usage of Dynamic Memory

The dynamic memory allocation is a core part of conventional C++. However, in 
some bare-metal products the usage of dynamic memory may be problematic and/or 
forbidden. The only way (I know of) to make to compilation fail, if dynamic 
memory is used, is to exclude standard library altogether. With `gcc` compiler 
it is achieved by using `-nostdlib` compilation option. 

Excluding standard library from the compilation will remove the whole C++ 
run-time environment, which includes dynamic memory (heap) management and 
exception handling. The implication of using this compilation option will be 
described later in [Removing Standard Library and C++ Runtime](nostdlib.md) section.
