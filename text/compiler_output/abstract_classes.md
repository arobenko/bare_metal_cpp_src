## Abstract Classes
The next thing to test is having abstract classes with pure virtual functions 
while excluding linkage to standard library (using `-nostdlib` compilation option). 
Below is an excerpt from 
[test_cpp_abstract_class](https://github.com/arobenko/embxx_on_rpi/tree/master/src/test_cpp/test_cpp_abstract_class) application.

```cpp
class AbstractBase 
{ 
public: 
    virtual ~AbstractBase(); 
    virtual void func() = 0; 
    virtual void nonOverridenFunc() final; 
}; 

class Derived : public AbstractBase 
{ 
public: 
    virtual ~Derived(); 
    virtual void func() override; 
}; 

AbstractBase::~AbstractBase() 
{ 
} 

void AbstractBase::nonOverridenFunc() 
{ 
} 

Derived::~Derived() 
{ 
} 

void Derived::func() 
{ 
} 
```

Somewhere in the “main” function:
```cpp
Derived obj; 
AbstractBase* basePtr = &obj; 
basePtr->func(); 
```

The compilation will fail with following errors:
```
CMakeFiles/04_test_abstract_class.dir/AbstractBase.cpp.o: In function `AbstractBase::~AbstractBase()': 
AbstractBase.cpp:(.text+0x24): undefined reference to `operator delete(void*)' 
CMakeFiles/04_test_abstract_class.dir/AbstractBase.cpp.o:(.rodata+0x10): undefined reference to `__cxa_pure_virtual' 
CMakeFiles/04_test_abstract_class.dir/Derived.cpp.o: In function `Derived::~Derived()': 
Derived.cpp:(.text+0x3c): undefined reference to `operator delete(void*)' 
```

The `__cxa_pure_virtual` is a function, address of which compiler writes in the 
virtual table when the function is pure virtual. It may be called due to some 
unnatural pointer abuse or when trying to invoke pure virtual function in the 
destructor of the abstract base class. The call to this function should never 
happen in the normal application run. If it happens it means there is a bug. 
It is quite safe to implement this function with infinite loop or some way to 
report the error to the developer, by flashing leds for example.
```cpp
extern "C" void __cxa_pure_virtual() 
{ 
    while (true) {} 
} 
```

The requirement for `operator delete(void*)` is quite strange though, there is no dynamic memory allocation in the source code. It has to be investigated. Let's stub the function and check the output of the compiler:
```cpp
void operator delete(void *) 
{ 
} 
```

The virtual tables for the classes reside in `.rodata` section:
```
Disassembly of section .rodata: 

000081a0 <_ZTV12AbstractBase>: 
	... 
    81a8:	000080d8 	ldrdeq	r8, [r0], -r8	; <UNPREDICTABLE> 
    81ac:	000080ec 	andeq	r8, r0, ip, ror #1 
    81b0:	0000815c 	andeq	r8, r0, ip, asr r1 
    81b4:	000080e8 	andeq	r8, r0, r8, ror #1 

000081b8 <_ZTV7Derived>: 
	... 
    81c0:	00008110 	andeq	r8, r0, r0, lsl r1 
    81c4:	00008130 	andeq	r8, r0, r0, lsr r1 
    81c8:	0000810c 	andeq	r8, r0, ip, lsl #2 
    81cc:	000080e8 	andeq	r8, r0, r8, ror #1 
```

The last entry for both classes has the address of `AbstractBase::nonOverridenFunc` function:
```
000080e8 <_ZN12AbstractBase16nonOverridenFuncEv>: 
    80e8:	e12fff1e 	bx	lr 
```

The third entry in the virtual table of **Derived** class has the address of 
`Derived::func` function, while the third entry in the virtual table of 
**AbstractBase** class has the address of `__cxa_pure_virtual`, 
just like expected.
```
0000810c <_ZN7Derived4funcEv>: 
    810c:	e12fff1e 	bx	lr 

0000815c <__cxa_pure_virtual>: 
    815c:	eafffffe 	b	815c <__cxa_pure_virtual> 
```

The first two entries in the virtual tables point to two different 
implementations of the destructor. The first entry has the address of normal 
destructor implementation, and the second one has an address of the second 
destructor implementation, that invokes operator delete 
(has `_ZdlPv` symbol) after the destruction of the object:
```
000080d8 <_ZN12AbstractBaseD1Ev>: 
    80d8:	e59f3004 	ldr	r3, [pc, #4]	; 80e4 <_ZN12AbstractBaseD1Ev+0xc> 
    80dc:	e5803000 	str	r3, [r0] 
    80e0:	e12fff1e 	bx	lr 
    80e4:	000081a8 	andeq	r8, r0, r8, lsr #3 

000080ec <_ZN12AbstractBaseD0Ev>: 
    80ec:	e59f3014 	ldr	r3, [pc, #20]	; 8108 <_ZN12AbstractBaseD0Ev+0x1c> 
    80f0:	e92d4010 	push	{r4, lr} 
    80f4:	e1a04000 	mov	r4, r0 
    80f8:	e5803000 	str	r3, [r0] 
    80fc:	eb000015 	bl	8158 <_ZdlPv> 
    8100:	e1a00004 	mov	r0, r4 
    8104:	e8bd8010 	pop	{r4, pc} 
    8108:	000081a8 	andeq	r8, r0, r8, lsr #3 

00008110 <_ZN7DerivedD1Ev>: 
    8110:	e59f3014 	ldr	r3, [pc, #20]	; 812c <_ZN7DerivedD1Ev+0x1c> 
    8114:	e92d4010 	push	{r4, lr} 
    8118:	e1a04000 	mov	r4, r0 
    811c:	e5803000 	str	r3, [r0] 
    8120:	ebffffec 	bl	80d8 <_ZN12AbstractBaseD1Ev> 
    8124:	e1a00004 	mov	r0, r4 
    8128:	e8bd8010 	pop	{r4, pc} 
    812c:	000081c0 	andeq	r8, r0, r0, asr #3 

00008130 <_ZN7DerivedD0Ev>: 
    8130:	e59f301c 	ldr	r3, [pc, #28]	; 8154 <_ZN7DerivedD0Ev+0x24> 
    8134:	e92d4010 	push	{r4, lr} 
    8138:	e1a04000 	mov	r4, r0 
    813c:	e5803000 	str	r3, [r0] 
    8140:	ebffffe4 	bl	80d8 <_ZN12AbstractBaseD1Ev> 
    8144:	e1a00004 	mov	r0, r4 
    8148:	eb000002 	bl	8158 <_ZdlPv> 
    814c:	e1a00004 	mov	r0, r4 
    8150:	e8bd8010 	pop	{r4, pc} 
    8154:	000081c0 	andeq	r8, r0, r0, asr #3 

00008158 <_ZdlPv>: 
    8158:	e12fff1e 	bx	lr 
```

It seems that when there is a virtual destructor, the compiler will have to 
support direct invocation of the destructor as well as usage of operator delete. 
In case of the former the compiler will use the first entry in the virtual table 
for the destructor invocation, and in case of the latter the compiler will use 
the second entry. Let's try to add the following lines to our `main` function:
```cpp
basePtr->~AbstractBase(); 
delete basePtr;
```

The compiler will add the following instructions to the `main` function:
```
    8190:	e59d3004 	ldr	r3, [sp, #4] 
    8194:	e1a00004 	mov	r0, r4 
    8198:	e5933000 	ldr	r3, [r3] 
    819c:	e12fff33 	blx	r3 
    81a0:	e59d3004 	ldr	r3, [sp, #4] 
    81a4:	e1a00004 	mov	r0, r4 
    81a8:	e5933004 	ldr	r3, [r3, #4] 
    81ac:	e12fff33 	blx	r3 
```

The address of the virtual table is written into **r3**, then value of **r3** 
is overwritten with address of the destructor function to call, and the call is 
executed using `blx` instruction. The first invocation takes the address of 
destructor function from the first entry of virtual table, while the second 
invocation takes the address from second entry (offseted by `#4`). 
This is just like expected.

**CONCLUSION**: Having virtual destructor may require an implementation of 
`operator delete(void*)` even if there is no dynamic memory allocation.

