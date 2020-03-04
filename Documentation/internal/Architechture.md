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

## 2.3 Implemention of some delve features

# 3 Reference
1. [Architecture of Delve slides](https://speakerdeck.com/aarzilli/internal-architecture-of-delve).