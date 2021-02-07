## 64位汇编
#### 函数调用：

当参数少于7个时， 参数从左到右放入寄存器: **rdi, rsi, rdx, rcx, r8, r9**。
当参数为7个以上时， 前 6 个与前面一样， 但后面的依次从 “右向左” 放入栈中，即和32位汇编一样。

参数个数大于 7 个的时候
H(a, b, c, d, e, f, g, h);
a->%rdi, b->%rsi, c->%rdx, d->%rcx, e->%r8, f->%r9
h->8(%esp)
g->(%esp)
call H

## Stack
Recall that the stack is a segment of memory used to store objects with automatic lifetime. Typical stack addresses on x86-64 look like 0x7ffd'9f10'4f58—that is, close to 247.

The stack is named after a data structure, which was sort of named after pancakes. Stack data structures support at least three operations: push adds a new element to the “top” of the stack; pop removes the top element, showing whatever was underneath; and top accesses the top element. Note what’s missing: the data structure does not allow access to elements other than the top. (Which is sort of how stacks of pancakes work.) This restriction can speed up stack implementations.

Like a stack data structure, the stack memory segment is only accessed from the top. The currently running function accesses its local variables; the function’s caller, grand-caller, great-grand-caller, and so forth are dormant until the currently running function returns.

x86-64 stacks look like this:

![Stack](https://cs61.seas.harvard.edu/site/img/stack-2018-01.png)

The x86-64 %rsp register is a special-purpose register that defines the current “stack pointer.” This holds the address of the current top of the stack. On x86-64, as on many architectures, stacks grow down: a “push” operation adds space for more automatic-lifetime objects by moving the stack pointer left, to a numerically-smaller address, and a “pop” operation recycles space by moving the stack pointer right, to a numerically-larger address. This means that, considered numerically, the “top” of the stack has a smaller address than the “bottom.”

This is built in to the architecture by the operation of instructions like pushq, popq, call, and ret. A push instruction pushes a value onto the stack. This both modifies the stack pointer (making it smaller) and modifies the stack segment (by moving data there). For instance, the instruction pushq X means:
```
subq $8, %rsp
movq X, (%rsp)
```
And popq X undoes the effect of pushq X. It means:
```
movq (%rsp), X
addq $8, %rsp
```
X can be a register or a memory reference.

The portion of the stack reserved for a function is called that function’s stack frame. Stack frames are aligned: x86-64 requires that each stack frame be a multiple of 16 bytes, and when a callq instruction begins execution, the %rsp register must be 16-byte aligned. This means that every function’s entry %rsp address will be 8 bytes off a multiple of 16.

## Return address and entry and exit sequence
The steps required to call a function are sometimes called the entry sequence and the steps required to return are called the exit sequence. Both caller and callee have responsibilities in each sequence.

To prepare for a function call, the caller performs the following tasks in its entry sequence.

1. The caller stores the first six arguments in the corresponding registers.

2. If the callee takes more than six arguments, or if some of its arguments are large, the caller must store the surplus arguments on its stack frame. It stores these in increasing order, so that     
    the 7th argument has a smaller address than the 8th argument, and so forth. The 7th argument must be stored at (%rsp) (that is, the top of the stack) when the caller executes its callq
    instruction.

3. The caller saves any caller-saved registers (see below).

4. The caller executes callq FUNCTION. This has an effect like pushq $NEXT_INSTRUCTION; jmp FUNCTION (or, equivalently, subq $8, %rsp; movq $NEXT_INSTRUCTION, (%rsp); jmp FUNCTION), where NEXT_INSTRUCTION is the address of the instruction immediately following callq.

This leaves a stack like this:
![Initial stack at start of function](https://cs61.seas.harvard.edu/site/img/stack-2018-02.png)

To return from a function:

1. The callee places its return value in %rax.

2. The callee restores the stack pointer to its value at entry (“entry %rsp”), if necessary.

3. The callee executes the retq instruction. This has an effect like popq %rip, which removes the return address from the stack and jumps to that address.

4. The caller then cleans up any space it prepared for arguments and restores caller-saved registers if necessary.

Particularly simple callees don’t need to do much more than return, but most callees will perform more tasks, such as allocating space for local variables and calling functions themselves.

## Callee-saved registers and caller-saved registers
The calling convention gives callers and callees certain guarantees and responsibilities about the values of registers across function calls. Function implementations may expect these guarantees to hold, and must work to fulfill their responsibilities.

The most important responsibility is that certain registers’ values must be preserved across function calls. A callee may use these registers, but if it changes them, it must restore them to their original values before returning. These registers are called callee-saved registers. All other registers are caller-saved.

Callers can simply use callee-saved registers across function calls; in this sense they behave like C++ local variables. Caller-saved registers behave differently: if a caller wants to preserve the value of a caller-saved register across a function call, the caller must explicitly save it before the callq and restore it when the function resumes.

On x86-64 Linux, %rbp, %rbx, %r12, %r13, %r14, and %r15 are callee-saved, as (sort of) are %rsp and %rip. The other registers are caller-saved.



## Base pointer (frame pointer)
The %rbp register is called the base pointer (and sometimes the frame pointer). For simple functions, an optimizing compiler generally treats this like any other callee-saved general-purpose register. However, for more complex functions, %rbp is used in a specific pattern that facilitates debugging. It works like this:

![Stack frame with base pointer](https://cs61.seas.harvard.edu/site/img/stack-2018-03.png)

1. The first instruction executed on function entry is pushq %rbp. This saves the caller’s value for %rbp into the callee’s stack. (Since %rbp is callee-saved, the callee must save it.)

2. The second instruction is movq %rsp, %rbp. This saves the current stack pointer in %rbp (so %rbp = entry %rsp - 8).
   This adjusted value of %rbp is the callee’s “frame pointer.” The callee will not change this value until it returns. The frame pointer provides a stable reference point for local variables 
   and caller arguments. (Complex functions may need a stable reference point because they reserve varying amounts of space for calling different functions.)
   Note, also, that the value stored at (%rbp) is the caller’s%rbp, and the value stored at 8(%rbp) is the return address. This information can be used to trace backwards through callers’ 
   stack frames by functions such as debuggers.

3. The function ends with movq %rbp, %rsp; popq %rbp; retq, or, equivalently, leave; retq. This sequence restores the caller’s %rbp and entry %rsp before returning.

## 寻址方式：

以前经常要背诵的什么基址寻址，变址寻址，基址变址寻址神马的太绕人了，又不好记。 我觉得内存地址的引用就这一种格式：

地址或偏移 (%基址或偏移量寄存器, %索引寄存器, 比例因子)

地址的计算公式是：

最终地址 = 地址或偏移 + %基址或偏移量寄存器 + %索引寄存器  * 比例因子

## 参考
https://cs61.seas.harvard.edu/site/2018/Asm2/
http://abcdxyzk.github.io/blog/2012/11/23/assembly-args/
