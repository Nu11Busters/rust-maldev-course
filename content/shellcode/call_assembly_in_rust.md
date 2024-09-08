+++
title = "Call assembly in Rust"
weight = 1
+++

Please note!

It is important to note that this course requires a basic knowledge of assembler concepts. Understand the structure of an assembler instruction, registers, memory, etc.

## Calling assembler in Rust

Assembler is a low-level language that represents the instructions directly executed by a computer's processor. Each instruction generally corresponds to a specific hardware operation.

### The benefits of assembler with Rust

Writing assembler code in Rust is interesting for a number of reasons. First of all, for development purposes, there's great interest, as you can manipulate low-level instructions while taking advantage of Rust's modern features. Rust's clear, expressive syntax simplifies the writing of assembler instructions, making it more accessible even to those who are not assembler experts. But also, shellcodes can be written in a more structured way, improving code maintenance over time. Of course, there are many other advantages to using Rust for assembler, depending on your needs. To summarize, assembler in Rust offers a balanced approach to leveraging the power of assembler while mitigating the challenges associated with using it directly.

### How to call assembler in Rust?

To illustrate my point, let's take the following assembler code as an example:
```
section .data
   message db 'Hello world!', 0
   len equ $ - message

section .text
   global _start

_start:

   ; Load syscall number sys_write (1) into rax
   mov rax, 1

   ; Load file descriptor for stdout (1) into rdi
   mov rdi, 1

   lea rsi, [message]
   mov rdx, len

   ; System call sys_write
   syscall

   ; End of program - Load sys_exit syscall number (60) into rax
   mov rax, 60

   Load output code 0 into rdi
   xor rdi, rdi

   ; System call sys_exit
   syscall
```

This code displays :

```Hello world!```

In the console, type the command :

```nasm -f elf64 hello.asm -o hello.o && ld hello.o -o hello && ./hello```

Next, you'll need to change the Rust compiler version from stable to nightly. Because the :

```asm!```

is currently experimental and requires the nightly version of the Rust compiler. To do this, simply take the Optimizing Cargo course.

Here's an example of how to configure Cargo.toml:
```
[package] name = "hello
version = "0.1.0
edition = "2021"

[dependencies]

[profile.release]
opt-level = "s"
strip = true
lto = true
panic = "abort
```

Now we can move on to the main part of this course, calling assembler in Rust. We're going to take the assembler code we saw earlier and integrate it into our Rust code. Here's the basis of our program:
```
use std::arch::asm;

fn main() {
   //Declaration of the string to be displayed in the console
   let message = String::from("Hello world!\n");
   unsafe {
      asm!(
         //Your assembler code will be here
      )
   }
}
```

In Rust, arguments can be called up in a different order than in assembler. In our case, the arrangement of arguments for a system call is :
```syscall```

may vary according to the system API convention used. In the case of the Linux x86-64 system API, the arguments for syscall :
```write```

are generally passed in the rdi, rsi, rdx, r10, r8 and r9 registers, in that order.
In our assembly code, the order of the arguments conforms to this convention:
```
Load syscall number sys_write (1) into rax
mov rax, 1
; Load file descriptor for stdout (1) into rdi
mov rdi, 1
; Load message pointer into rsi
lea rsi, [message]
; Load message length into rdx
mov rdx, len
; System call sys_write
syscall
```

In assembly code, loading the pointer to the message :

```lea rsi, [message]```

And the length of the message:

```mov rdx, len```

Performed before the system call

```syscall```

In accordance with Linux x86-64 convention.

However, the situation is different in Rust code. In the case of the :

```asm!```

Constraints:

```in```

Specify operands that are inputs to the assembly instruction, but this does not necessarily determine the order of evaluation. In your Rust code, in constraints are specified after syscall:
```
asm!(
   "mov rax, 1",
   "mov rdi, 1",
   "syscall",
   in("rsi") message.as_ptr(),
   in("rdx") message.len(),
)
```
The evaluation order can change depending on the compiler and the optimizations performed. This can sometimes lead to a change in the apparent order of in constraints compared to assembly code. If we take a closer look :

```"mov rax, 1"```

 This instruction loads the syscall number into rax.

```"mov rdi, 1"```

 This instruction loads the stdout file descriptor into rdi.

```"syscall"```

This instruction performs the syscall write. At this point, the values in rsi and rdx will be used as parameters for syscall.

```in("rsi") message.as_ptr()```

This constraint specifies that rsi is an input and loads the address of the beginning of the message.as_ptr() string into rsi. This is important because rsi is an input register for syscall.

```in("rdx") message.len()```

This constraint specifies that rdx is an input and loads the length of the message.len() string into rdx. It's also crucial because rdx is used as an input parameter for syscall.

However, as long as the constraints are correctly specified and the values are correctly loaded into the registers before syscall, the result should be correct. It's important to note that specifying in constraints after syscall in Rust code doesn't affect the actual order in which operations are evaluated, and the compiler will ensure that the necessary values are correctly loaded before the system call.
So here's our final program:
```
use std::arch::asm;

fn main() {
   let message = String::from("Hello world!\n");
   unsafe {
      asm!(
         "mov rax, 1",
         "mov rdi, 1",
         "syscall",
         in("rsi") message.as_ptr(),
         in("rdx") message.len(),
      )
   }
}
```

Finally, we can see that calling assembler in Rust is pretty easy. You just need to pay attention to the evaluation order and the conventions of the system APIs. To play with variables, use the various operands available in Rust:

```in, out, inout, inlateout, lateout, etc.```

The in operand is used to specify that the values of the specified registers will be read by the assembler instruction. This means that the register values will be used as input for the assembler instruction.
Example:

```asm!("mov $0, $1" : "=r"(output) : "r"(input));```

The out operand is used to specify that the specified register values will be written by the assembler instruction. This means that the register values will be modified by the assembler instruction and returned as output.
Example :

```asm!("mov $0, $1" : "=r"(output) : "r"(input));```

The inout operand combines in and out. It specifies that the values of specified registers will be read as input and written as output by the assembler instruction.
Example:

```asm!("add $0, $1" : "=r"(result) : "r"(value));```

The inlateout operand is similar to inout, but indicates that the write operation (out) can be delayed until the end of the asm! block.
Example:

```asm!("add $0, $1" : "=&r"(result) : "r"(value));```

The lateout operand is used to specify that the write operation (out) can be delayed until the end of the asm! block, without any initial read operation.
Example:

```asm!("mov $0, $1" : "=&r"(output));```

In these examples, $0, $1, etc. are substitution operands used to reference the register values specified in the operand list. The symbols "=r" and "r" define the type and access mode of the registers.

All the code seen in this course is available as a file on the following page.
