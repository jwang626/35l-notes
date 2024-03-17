#W10L1
# Introduction

So far we've seen debugging, which is a type of dynamic analysis. 
- think of GDB as a program execution explorer (tests all the ways the program can be run)

But debugging is part of a much broader topic, which is analysis to address issues that may not be explicit bugs.

## Difference between static and dynamic?

**Static analysis** involves examining code at compile time without actually executing it. This can identify areas of optimization, security vulnerabilities, and compile-time errors.
- pros: early detection (can eliminate entire classes of errors), doesn't require execution, automated, scalability
- cons: typically require access to source code. many false positives, limited context+ detection of runtime errors, complex configurations. aka not as precise as dynamic

**Dynamic analysis** evaluates the program during its execution. It can catch issues that static missed, like runtime errors, memory leaks, and performance bottlenecks.
- pros: doesn't need source code (good for industry). real-time behavior, context-rich analysis, debugging support
- cons: limited code coverage (only one execution path), overhead, limited scalability

## Why do we do this?

1. Performance issues
2. Security issues

We will now talk about this in terms of C Programming and `gcc` compiler, but it is the same idea for all languages.

# Performance 

First, we will talk about methods that happen at compile time (compiler generated optimizations) aka STATIC ANALYSIS.

## Most Basic Method: GCC optimization flags

`gcc -O1` all the way to `-O4`
Recall: higher optimization = ideally a faster/smaller program.
Butt, we have these issues:
- slower `gcc` 
- harder to debug
- sometimes the program gets bigger (more code generated)
- machine dependent- more code could blow instruction cache and actually make code go slower!!

Lesson: We need to measure the results of optimization.
- `$ size` gives size of file
- `$ time` times the program

Or, we could write a more efficient algorithm. But what if this doesn't work? How else can we give advice to the compiler to optimize?

Let's talk about some other methods.

## The unreachable() function (Static)

`#include <stddef.h>`
- Semantics: this unreachable function must never be called in the program. if called, behavior is undefined
- can be used to indicate parts of the code that should never be executed

```C
#include <stddef.h>
void divide_by_four(int value){
	if (value<0){
		unreachable();
	}
	else {
		value >> 2;
	}
}
```
- can package this function into a macro

<ins>Why does Unreachable() improve performance?</ins>

In the entire if (condition) then unreachable(), it generates NO machine code! This is because unreachable() marks this branch as will not be taken so compiler will omit generating code for entire branch. Whereas other methods like conditional checking, assert statements generate more code.

## Cold and Hot Attributes (Static)

`__attribute__((hot))` and `__attribute__((cold))`

Cold and hot indicate how likely code is to be executed often.
Essentially advice to compiler on how to use instruction cache effectively.
- compiler might place cold code in a different section of the binary to improve cache usage for the more frequently executed parts of the code.

```C
__attribute__((cold))
void rarelyUsedFunction() {
}
```

Note that this has potential for human error (give compiler bad advice). Profiling addresses this.

## Noreturn attribute (Static)

`_noreturn`

Indicates to compiler that function will not return.
- exit and abort are examples that use this
```C
_noreturn void exit(int);
```

## Profiling (Dynamic)

This is a dynamic analysis tool that tells compiler to generate machine code to measures frequency of each executed instruction.
- good for large apps
`$ gcc --coverage` option allows us to generate profile.
Then you recompile with this source code + profile: `$ gcc fprofile-use` , and GCC will automatically figure out hot/cold spots (?)

Potential issues:
- profiling is approximate (might be different from run to run)
- slow
- overhead to store profile info

## Whole Program Optimization (Static)

Use `gcc -flto` (stands for **Link-Time Optimization**)
- happens during linking phase
- instead of optimizing individual object files separately and then linking them together, GCC performs optimizations on the combined code of all object files after they have been compiled.
- good for small apps. too slow for large apps.
- want to do this for programs that will execute often
Issues:
- expensive, slow (optimization is costly)

`gcc -flto a.c` , `gcc -flto b.c` puts a copy of source code into `.o` files,
then `gcc -flto a.o b.o -o myprog`

# Security (Dynamic Checking)

C being memory-unsafe leads to all kinds of security vulnerabilities.
Lets look at a couple of dynamic checking techniques (typically run slower since need to generate extra code, and is only specific to one run).

## Defense Against ROP/ Buffer Overflow Attacks 

The following gcc options are all methods of Dynamic Checking (during execution) that defend against **Return-Oriented Programming (ROP)**, which is a method used by attackers to take control of return instructions in your program.
- ROP repurposes existing code fragments and rewrites so it bypasses existing security measures like ASLR and non-executable memory

### Enable Stack Protection 

`gcc -fstack-protector`
- Generated for functions that are sus (declaring arrays) to prevent stack buffer overflow attacks
- Compiler will generate extra instructions that will push onto stack a random value called a stack canary just before the return instruction. Check if this value has changed (indicates return is overwritten/attacked). If so, `abort()`.
Cons:
- stack protector slows down execution since compiler added extra instructions

### Shadow Stack Protection 

`gcc -mshstack`
This is a newer feature, not available on all chips.
Pros:
- unlike stack-protector, doesn't slow down. Gets hardware support 
What is a **shadow stack**?
- We already have the ordinary/main stack with stack pointer rsp
- Have extra memory called shadow stack that's smaller than main stack and contains *only values of return addresses* (which are copies of what's in the main stack) 
- <ins>not writeable, except by the call instruction</ins> so it is protected against modifications
	- call instr will push return addresses onto both stacks
	- return instr will pop return address from both stacks, check to make sure the same. If not, then trap
Cons:
- trading memory (need extra memory for shadow stack) for security

## Defenses Against Other Issues

### stdckdint library for Integer Overflow

`#include <stdckdint.h>`
### GCC options

`-fsanitize=address`
- Tells GCC to generate more machine code to catch *most* memory violations (subscript errors, bad pointers)

`-fsanitize=undefined`
- Generates more machine code to cause program to *crash reliably* at every instance of undefined behavior

`-fsanitize=leak`
- detects memory leaks (forget to deallocate memory)
- difference from `address`: doesn't cause program to crash or cause undefined behavior, just hogs memory

`-fsanitize=thread`
- attempts to detect race conditions 

Cons of this method:
- options slow down the program, sometimes immensely.
- you need to recompile for each of these (compiler generates different executable code).

The alternative: Valgrind.

### Valgrind
- an interpreter that takes your normally-built executable and single-steps through program and looks for certain classes of errors (memory debugging, profiling)
	- for each sus instruction, checks value of register
- slower (interpreting) but no recompilation
- does NOT need source code, just looks at `.o` binary file
`valgrind --tool=memcheck ./myscript` : memcheck tool is most popular
# Security (Static Checking)

## static_assert

If condition evaluates to 0 (False), then refuse to compile.
```C
#include <assert.h>
static_assert(condition);
```
Pros:
- no runtime code, doesn't slow down
- works for all runs
Cons:
- only works on constant expressions (values have to be known statically at compilation)
- more work to change the source code

## GCC options (warnings)

`gcc -W` or `gcc -Wall` enables all static checking that GCC devs think is useful

`-Wall` implies
- `-Wcomment` (for bad comment style)
- `-Wparentheses` (for ambiguous expressions)
- `-Waddress` (ex for pointer comparison)
- `-Wstrict-aliasing` is controversial
	- accessing an object through a pointer of a different type than the one it was originally declared with is undefined behavior
```C 
int main() {
    int i = 10;
    float *fptr = (float*)&i;
    *fptr = 3.14;
    printf("%d\n", i);
    return 0;
}
```
	gcc -Wall -Wno-strict-aliasing
- `-Wmaybe-uninitialized` (only looks at code in each function)
- `-Wextra`
		- `-Wtype-limits` (related to comparisons that are always true or false due to the limited range of a data type)

More general option:

`gcc -fanalyzer` 
- looks thru "all" paths of program statically for common errors
- takes into the larger context of all functions etc (more powerful)
- very slow, esp with `-flto`
- many false positives (silenceable by disabling warnings)

### Disabling GCC Warnings

`gcc -w` disables all

`gcc -Wno-<warning>` will disable the specific `<warning>



