# 1 Internal Overview
Delve is a symbolic debugger for the Go Programming Language, used by Goland IDE, VSCode Go, vim-go, etc.
This document will give a general overview of delve's architecture, and explain why other debuggers have difficulties with Go programs.

# 2 Contents

## 2.1 Assembly Basics

### 2.1.1 CPU

- Computers have CPUs
- CPUs have registers, in particular:
    - "Program Counter" (PC), where stores the address of the next instruction to execute
        >also known as Instruction Pointer (IP)
    - "Stack Pointer" (SP), where stores the address of the "top" of the call stack
- CPUs execute assembly instructions that look like this:
    ```c
    MOVQ DX, 0x58(SP)
    ```
  
### 2.1.2 Call Stack

Normally, each function call has a isolated call stack frame, where stores the arguments, local variables and  return address of a function call.

Following is an illustration:

|||
|:--------------------:|:------------------:|
|Locals of runtime.main| <- high address    |
|Ret.address           |                    |
|Locals of main.main   |                    |
|Arguments of main.f   |                    |
|Ret.Address           |                    |
|Locals of main.f      | <- low address     |

Goroutine 1 starts by calling runtime.main:

|||
|:--------------------:|:------------------:|
|Locals of runtime.main|                    |
|                      | <- SP              |

runtime.main calls main.main by pushing a return address on the stack:

|||
|:--------------------:|:------------------:|
|Locals of runtime.main|                    |
|Ret.address           |                    |
|                      | <- SP              |

main.main pushes its' local variables on the stack:

|||
|:--------------------:|:------------------:|
|Locals of runtime.main|                    |
|Ret.address           |                    |
|Locals of main.main   |                    |
|                      | <- SP              |

Wheen main.main calls another function `main.f`:
 - it pushes the arguments of main.f on the stack
 - pushes the return value ono the stack
 
|||
|:--------------------:|:------------------:|
|Locals of runtime.main|                    |
|Ret.address           |                    |
|Locals of main.main   |                    |
|Arguments of main.f   |                    |
|Ret.Address           |                    |
|                      | <- SP              |

Finally main.f pushes its local variables on the stack:

|||
|:--------------------:|:------------------:|
|Locals of runtime.main|                    |
|Ret.address           |                    |
|Locals of main.main   |                    |
|Arguments of main.f   |                    |
|Ret.Address           |                    |
|Locals of main.f      |                    |
|                      | <- SP              |

Well, when calling a function, pushing the arguments, pushing the return address, saving the Base Pointer, assigning the value of Base Pointer to Stack Pointer... These actions are defined by ABI (Application Binary Interface).

### 2.1.3 Threads and Goroutines

- M:N threading / green threads
    - M goroutines are scheduled cooperatively on N threads
        > Well, go1.14 supports non-cooperatively preemption schedule in tight loop. 
    - N initially equal too $GOMAXPROCS (by default the number of CPU cores)
        > If Hyper-Threading supported, $GOMAXPROCS equal to number of hardware thread.
- Unlike threads, goroutines:
    - are scheduled cooperatively
        > Well, go1.14 supports non-cooperatively preemption schedule in tight loop, which is implemented by signal SIGURG. While thread preemption is implemented by hardware interrupt.
    - their stack starts small and grows/shrinks during execution
- When a go function is called
    - it checks that if there is enough space on the stack for its local variables
    - if the space is not enougth, runtime.morestack_noctx is called
    - runtime.morestack_noctx allocates more space for the stack
    - if the memory area below the current stack is already used, the stack is copied somewhere else in memory and then expanded
        > maybe adjusting the values of the pointers in current stack is needed
- Goroutine stacks can move in memory
    - debuggers normally assume stacks don't move
        > as mentioned above, goroutine stack can grow and shrink, it maybe moved here and there in memory, so the register SP must be changed to address the right position. while debuggers normally assume stacks don't move, so they may behave totally wrongly.

## 2.2 Architecture of Delve

### 2.2.1 Architecture of a Symbolic Debugger

|                |                   |
|:--------------:|:------------------|
| UI Layer       | the debugging user interface, like: <br>- command line interface<br>- graphical user interface like GoLand or VSCode Go.
| Symbolic Layer | knows about:<br> - line numbers, .debug_line<br>- types, .debug_types<br>- variable names, .debug_info, etc.
| Target Layer   | controls target process, doesn't know anything about your source code, like<br>- set breakpoint<br>- execute next statement<br>- step into a function<br>- step out a function<br>- etc.

### 2.2.2 Features of the Target Layer

- Attach/detach from target process
- Enumerate threads in the target process
- Can start/stop individual threads (or the whole process) 
    > CPU instruction patching, like X86 int3 generates 0xCC
- Receives "debug events" (thread creation/death and most importantly thread stop no a breakpoint)
- Can read/write the memory of the target process
    > like Linux ptrace peek/poke data in memory
- Can read/write the CPU registers of a stopped thead
    - actually this is the CPU registers saved in the thread descriptor of the OS scheduler
    > like Linux ptrace peek/poke data in registers.<br/> <br/>
    well, when thread is switched off the CPU, its hardware context must be saved somewhere. when the thread becomes runnable and scheduled, its hardware context saved before will be resumed. So where does this hardware context saved? please check the knowledge about GDT, it holds the thread descriptor entries and code segment priviledge control relevant entries.

For now, we have 3 implementions of the target layer:
- pkg/proc/native: controls target process using OS API calls, supports:
    - Windows  
    `WaitForDebugEvent`, `ContinueDebugEvent`, `SuspendThread`...
    - Linux
    `ptrace`, `waitpid`, `tgkill`...
    - macOS
    `notification/exception ports`, `ptrace`, `mach_vm_region`...
    > /pkg/proc/native, it's the default backend on Windows and Linux
- pkg/proc/core: reads linux_amd64 core files
- pkg/proc/gdbserial: used to connect to:
    - debugserver on macOS (default setup on macOS)
    - lldb-server
    - Mozillar RR (a time travel debugger backend, only works on linux/amd64)
    > the names comes from the protocol it speaks, the Gdb Remote Serial Protocol
 
About debuggserver
- pkg/proc/gdbserial connected to debugserver is the default target layer for macOS
- two reasons:
    - the native backend uses undocumented API and never worked properly
    - the kernel API used by the native backend are restricted and require a signed executable
        - distributing a signed executable as an open source project is problematic
        - users often got the self-signing process wrong

## 2.3 Implemention of some delve features

# 3 Reference
1. [Architecture of Delve slides](https://speakerdeck.com/aarzilli/internal-architecture-of-delve).