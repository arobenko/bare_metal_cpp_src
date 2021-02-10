## RTTI

**Run Time Type Information** is also one of the core features of conventional C++. It allows retrieval of the object type information (using [typeid](http://en.cppreference.com/w/cpp/language/typeid) operator) as well as checking the inheritance hierarchy (using [dynamic_cast](http://en.cppreference.com/w/cpp/language/dynamic_cast)) at run time. The RTTI is available only when there is a polymorphic behaviour, i.e. the classes have at least one virtual function.

Let's try to analyse the generated code when RTTI is in use. The [test_cpp_rtti](https://github.com/arobenko/embxx_on_rpi/tree/master/src/test_cpp/test_cpp_rtti) application in [embxx_on_rpi](https://github.com/arobenko/embxx_on_rpi) project contains the code listed below.
```cpp
struct SomeClass
{
    virtual void someFunc();
};
```

Somewhere in *.cpp file:
```cpp
void SomeClass::someFunc()
{
}
```

Somewhere in `main` function:
```cpp
SomeClass someClass;
someClass.someFunc();
```

Let's open the listing file and see what's going on in there. The address of `SomeClass::someFunc()` seems to be `0x8300`:
```
00008300 <_ZN9SomeClass8someFuncEv>:
    8300:	e12fff1e 	bx	lr
```

The virtual table for `SomeClass` class must be somewhere in `.rodata` section and contain address of `SomeClass::someFunc()`, i.e. it must have 0x8300 value inside:
```
Disassembly of section .rodata:

...
00009c10 <_ZTV9SomeClass>:
    9c10:	00000000 	andeq	r0, r0, r0
    9c14:	00009c04 	andeq	r9, r0, r4, lsl #24
    9c18:	00008300 	andeq	r8, r0, r0, lsl #6
    9c1c:	00000000 	andeq	r0, r0, r0

```

It is visible that compiler added some more entries to the virtual table in addition to the single virtual function we implemented. The address 0x9c04 is also located in `.rodata` section. It is some type related table:
```
00009c04 <_ZTI9SomeClass>:
    9c04:	00009c28 	andeq	r9, r0, r8, lsr #24
    9c08:	00009bf8 	strdeq	r9, [r0], -r8
    9c0c:	00000000 	andeq	r0, r0, r0
```

Both 0x9c28 and 0x9bf8 are addresses in `.rodata*` section(s). The 0x9bf8 address seems to contain some data:
```
00009bf8 <_ZTS9SomeClass>:
    9bf8:	6d6f5339 	stclvs	3, cr5, [pc, #-228]!	; 9b1c <strcmp+0x180>
    9bfc:	616c4365 	cmnvs	ip, r5, ror #6
    9c00:	00007373 	andeq	r7, r0, r3, ror r3
```
After a closer look we may decode this data to be "9SomeClass" ascii string.

Address 0x9c28 is in the middle of some type related information table:
```
00009c20 <_ZTVN10__cxxabiv117__class_type_infoE>:
    9c20:	00000000 	andeq	r0, r0, r0
    9c24:	00009c50 	andeq	r9, r0, r0, asr ip
    9c28:	00009dc0 	andeq	r9, r0, r0, asr #27
    9c2c:	00009de4 	andeq	r9, r0, r4, ror #27
    9c30:	0000a114 	andeq	sl, r0, r4, lsl r1
    9c34:	0000a11c 	andeq	sl, r0, ip, lsl r1
    9c38:	00009e40 	andeq	r9, r0, r0, asr #28
    9c3c:	00009d48 	andeq	r9, r0, r8, asr #26
    9c40:	00009e10 	andeq	r9, r0, r0, lsl lr
    9c44:	00009e94 	muleq	r0, r4, lr
    9c48:	00009dac 	andeq	r9, r0, ip, lsr #27
    9c4c:	00000000 	andeq	r0, r0, r0
```

How these tables are used by the compiler is of little interest to us. What is interesting is a code size overhead. Lets check the size of the binary image. With my compiler it is a bit more than 13KB. 

For some bare metal platforms it may be undesirable or even impossible to have this amount of extra binary code added to the binary image. The GNU compiler (`gcc`) provides an ability to disable **RTTI** by using `-no-rtti` option. Let's check the virtual table of `SomeClass` class when this option is used:
```
Disassembly of section .rodata:

00008320 <_ZTV9SomeClass>:
	...
    8328:	00008300 	andeq	r8, r0, r0, lsl #6
    832c:	00000000 	andeq	r0, r0, r0
```

The virtual table looks much simpler now with single pointer to the `SomeClass::someFunc()` virtual function. There is no extra code size overhead needed to maintain type information. If the application above is compiled without exceptions (using `-fno-exceptions` and `-fno-unwind-tables`) as well as without RTTI support (using `-no-rtti`) the binary image size will be about 1.3KB which is much better.

However, if `-no-rtti` option is used, the compiler won't allow usage of 
[typeid](http://en.cppreference.com/w/cpp/language/typeid) operator as well as 
[dynamic_cast](http://en.cppreference.com/w/cpp/language/dynamic_cast). 
In this case the developer needs to come up with other solutions to differentiate 
between objects of different types (but having the same 'ancestor') at run time. 
There are multiple idioms that can be used, such as using simple C-like approach 
of `switch`-ing on some type enumerator member, or using polymorphic behaviour 
of the objects to perform 
[double dispatch](http://en.wikipedia.org/wiki/Double_dispatch).

**CONCLUSION**: Disabling **R**un **T**ime **T**ype **I**nformation (RTTI) 
in addition to eliminating exception handling is very common in bare metal 
C++ development. It allows to save about 10KB of space overhead in final binary 
image.
