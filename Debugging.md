#W9L1 #W9L2
# First, Some General Strategies and Tactics

1. Don't. It's inefficient way to find and fix bugs
	- prevent bugs from happening when possible (static checking)
	- make them easy to find when they do happen (dynamic checking)
2. Test Cases (TDD: Test Driven Development)
	- need to maintain them
3. Use better platform (Rust, Java, Python, not C/C++)
4. **Defensive Programming**

## Here are some defensive techniques: 

1. runtime checking of your own
2. assert macro (not that the expression should be side-effect free)
	- if false, triggers assertion failure which prints diagnostic message and calls abort 
	- compile with assertions enabled: `gcc -DNDEGUB`	
``` C
#include <assert.h>
assert (some expression)
```
3. Exception handling 
4. Traces and Logs
	- print statements
	- `strace`
1. Checkpoint + restart functions
	- every so often, call checkpoint() which returns a file with program state. Then then feed this file into restart(file) which returns this back into the program 
	- can restart from most recent checkpoint
2. Barricades
	- given a big messy program that is fed outside data, focus on innermost core and clean up first (reliable data structures)
	- set up a barricade between messy part and clean core
		- API checks data that it lets into core
	- goal is to migrate the barricade outward (expand portion of clean code)
3. Interpreters (barricades on steroids)
	- not only bad data, but bad code coming in
		- ex: browser like Chrome. Chrome has a JS interpreter that controls the outside code run
	- the interpreter scrutinizes the bad code before letting it at our program, the browser
4. Virtual Machines (interpreters on steroids)
	- VMs are software-based representations of physical computers that run an OS and applications
	- create isolated environments for running entire applications

## Definitions

**Failure**: program behavior is wrong, user-visible problem
**Fault**: latent bug in program that could cause failure in some cases
**Error**: mental mistake by developer 

Errors don't have to lead to faults (ex: incorrect comment describing code) and faults don't have to lead to failures (faulty code never executed). We are trying to fix failures with debuggers (talked about fixing faults/errors previously).

## Approach

Don't guess at random, need systematic approach:
1. stabilize the failure (reproducible bug)
2. locate failure's source
3. fix the bug

Note: We often spend most time in the earlier steps.

Tools to do this:
1. print statements (generate logs)
2. GDB

Aside on segfaults:
Accessing invalid memory address. Note that region around 0 and around 1 is always invalid.

# GDB 

## Overview of GDB 

Why GDB?
- low-level debugger
- oldest debugger, lots of features

What is GDB?
- a process (a program in execution) that takes control of your processes

In this model, there are *two separate programs* (GDB and your program) here to insulate GDB from your buggy program (want to just insulate one way)

GDB Big Picture:
- stop/freeze your process
- read process memory + registers, including rip
- while process is stopped, can write to all the above
- then continue executing your process

Note: since GDB can modify instruction pointer, it can completely change what code is being run.

## Getting Started with GDB
### -g and -O flags

`gcc -g` generates debugging information and puts it into object files and executables (now have bigger files as overhead)
- info contains: 
	- symbol table (var, function names, types, memory addresses)
	- local, global, static variables
	- macro defs
- different levels of `-g` flag (higher levels result in larger files)

`gcc -O` changes optimization level + machine code, which affects performance
- higher levels of `-O` flag get more aggressive optimizations, but these are harder to debug
	- ex: local variable values are discarded, loop unrolling, function inlining

The problem of disabling optimization while debugging is that **you are essentially debugging a totally different program**. So what to do?

`gcc -Og` means optimize somewhat for debuggability.

## GDB Commands 

### Quick Start- Running GDB process itself

Normally we run debugger under an IDE but lets do it in the shell.

`$ gdb` shows `(gdb)` , exit using `(gdb) exit`.

To run GDB in emacs: `$ gdb emacs` 

To feed GDB a program (note this will NOT run the program), do `gdb ./program_name`

**To set any command line arguments** that your program needs upon start, use `(gdb) set args`. (Ex: `set args parameter1 parameter2`)

#### ASLR (Address Space Layout Randomization)

This picks a random location in memory to allocate new storage when malloc called. Used to defend against attackers. This causes problems in debugging (stabilizing failure).

`(gdb) set disable-randomization on` to disable randomization

So far, we don't have a subsidiary process running yet. Only GDB is running.

### Run your own process under GDB 

Use `run`: `(gdb) run my_process.txt` 

OR with `attach` and `detach` : `(gdb) attach process_pid`
- GDB will take over this process and stop it
- has to be YOUR process, have your userID

Attach is very powerful. What happens if you try to attach GDB to your login shell, or worse, GDB itself?
- normally there are safeguards in place to disable this. Linux: `ptrace`
- recursive debugging loop, deadlock

### Examine State of Program

**Backtrace**: `(gdb) bt` . 
- looks at runtime stack of program, how did I get to where I am now?
- will NOT give complete set of information, only gives info that program keeps around normally

**Print**: `(gdb) print my_var`
- which variable are we talking about? the one that is relevant to current context
- formatting: `print/d` decimal , `print/x` hex,  `print/s` string, `@10` displays first 10 elements of array
- can give print an arbitrary expression, ex: `(gdb) p a[10]+b`
- can even call user defined functions!! GDB arranges for subsidiary process to call the function
	- ex: `(gdb) p my_func(5)`
	- ex: `(gdb) p x=y` 
	- hijacked process by setting instructing pointer to my_func() and breakpoint at the end
**Potential Issues with Print:**
- *State Modification*- If function modifies global or static variables, alters data structures, writes to files, or has other side effects, these changes will persist in the program's state, potentially affecting its behavior when execution resumes.
- *Memory/Resource Allocation*: If  function allocates memory and there's no corresponding deallocation within the same function call, could lead to memory leaks.

**Change context Up/Down**: `(gdb) up` and `(gdb) down`
- for ex: if you want to examine variables in other function on stack
- **up** moves you closer to top of stack (earlier function calls/start of program) like main
- **down** moves you to be near bottom of stack (more recent calls)

## Extending GDB Functionality

We need ability to configure debugging tools. We can do this at the  GDB level or Python level.

### GDB Configuration File

`.gdbinit` placed in `~/.gdbinit` by default, but if there is a init file in the CURRENT directory it will override one in home (ex: `emacs/src/.gdbinit)` 

When GDB starts up, it configures itself using the file located in the current directory. We can define custom commands, set default arguments, automate repetitive tasks, run Python scripts.

Syntax: standard GDB commands or GDB scripting syntax.

### Custom Commands & GDB Scripting Syntax

**Define** a command
```
define <command>
    <code>
end
```

**Document** a command (appears when `(gdb) help commandname` is run)
```
document <command>
    <information about the command>
end
```

If the custom command takes **arguments** (ex: `(gdb) <command> <arg0> <arg1> <arg2>`):
- `$argc` total count, `$arg0`, `$arg1` ...

**Variables**:
- To store local variables: `$<variable_name>`
- set values with `set $<variable_name> = <value_or_expression>`

Conditionals and While Loops
```
if <condition>
    <code>
else
    <code>
end
```

```
while <condition>
    <code>
end
```

**Printing** (similar to printing in C)
- `printf "<format string>", <arg0>, <arg1>, ...`
- `print <expr>`: Evaluate the expression expr and display the resulting value

**Calling** functions:
- `call <expr>` : Evaluate the expression expr without displaying void returned values

Example run with `(gdb) printargs arg1Value arg2Value` :
```
define printargs
    # Print the number of arguments
    printf "Number of arguments: %d\n", $argc
    
    # Check if there are any arguments and print them
    if $argc > 0
        printf "Argument 0: %s\n", $arg0
    end
    if $argc > 1
        printf "Argument 1: %s\n", $arg1
    end
    # Add more checks if needed for more arguments
end
```

Lecture example: print the value of the long pointer
```
define pl
	print *(long*) $arg1
end
```

Note that GDB supports Python Scripting.

## Remote Debugging

So far, we talked about GDB debugging process running on own machine. But what if you need it to debug a remote application?

GDB process running on Machine A, subsidiary my_process running on Machine B.
- connected by some sort of wire (more latency)
Potential issues:
- different machine architectures
There's a whole set of commands in GDB to deal with remote debugging. `(gdb) target` 

## Breakpoints

<ins>General process:</ins>
1. Set a bp
2. Run the process (will stop at bp)
3. Examine state at this stopped point
4. Continue with `cont`

### Helpful Breakpoint Commands

`(gdb) info breakpoints` lists info about all bps with number, location
`(gdb) delete bp_num`  deletes the bp with specific bp number
`(gdb info registers` 
temporarily `disable` or `enable`

**Breakpoint**: `(gdb) break location` 
- *breakpoints apply to instruction pointer*
- location can be function name, file name with optional line number, address in memory (ex: `break main.cpp:55`)
- stops when reached beginning of location
- stay until you quit GDB and start it 

**Variations on Breakpoints:**
Single-stepping
- `step` breaks after every source code line
- `stepi` breaks after one machine instruction
- `next` steps over function calls and goes to next line in current function
- `finish` stops when current function returns 

*How does GDB implement breakpoint?*
- Replace first byte of target instruction with a "trap instruction" to make it stop. 
	- note: we're only changing machine instructions, not the program data (located in diff parts of memory)
- If you want to continue, GDB will restore original instruction so that you can move past the bp. Once you are past the bp, GDB will re-insert the trap instruction so that bp is still there

## Other: Checkpoints, Watchpoints

**Reverse Continue:** `(gdb) rc`
- GDB runs program backwards thorugh history of execution until hits bp (ex: assignments are undone)
- useful to figure out point of failure

*How does GDB implement this??*
- must start up GDB with special flag
- GDB single-steps program and remembers in GDB's own memory the old contents of program (keeps internal log) so that it can undo everything by taking log
- very slow

Alternative to RC that isn't so expensive: restart from checkpoints. This is particularly helpful if the program has a super long execution time and a crash happens like 30 minutes into it.

**Checkpoints**: `(gdb) info checkpoints` `(gdb) checkpoint` , `(gdb) restart chkpt_num`, `(gdb) delete chkpt_num`
- GDB saves snapshot of current process state in GDB's own memory so it can restore to it later
- can take checkpoints periodically, less expensive than RC

**Watchpoints**: `(gdb) watch`
- *watchpoints apply to data*
- stops when data value changes
- there is hardware support for watchpoints at full speed










