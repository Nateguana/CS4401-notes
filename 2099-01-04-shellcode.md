---
title:  "Lecture Notes: The Basics of Shellcoding"
date:   2020-01-04 03:00:00
categories: notes lecture
layout: post
challenges: stack4 stack5 stack6
---

If, as an attacker, you can't overwrite/modify the existing code for a process,
then the logical next step to supply your own malicious code. This class of
attack is generally referred to as **code injection**. When that code launches
a shell (giving you a convenient interface to the hacked machine), that injected
code is called **shellcode.** 

Understanding how to write shell code is what the latter sections of "Smashing
the Stack" are about. The paper not only describes how to write the shellcode,
but how to inject shell code and then trick the process into executing that
shellcode---with both steps enabled by the buffer overflow.

Writing shell code can be tricky at first because it requires knowledge of
assembly, C calling conventions, and the calling convention for system calls.

### Writing Shellcode

The author of "Smashing the Stack" took a simple approach to writing shellcode.
First, you write a simple C program that does what you want (in this case it
launches a shell using the `execve` system call). Second, you disassemble that
program to see how the desired behavior was implemented using machine
instructions. Third, extract, combine, and modify those instructions to form
your shellcode string.  This string is then injected into a vulnerable program.

#### The `execve` system call

Think of a system call as a means for a user-mode process to give control to
the  OS kernel to accomplish some task on the process's behalf.  See your
favorite OS textbook for a quick refresher. 

In the context of our shellcode, we want to use a system calls to launch a
shell that we can interact with. In particular, we are going to use  the
`execve` system call. If `execve` succeeds, it will transform the current
process into something new.  That "something new" is determined by the
arguments the programmer passes to exec. Thus, if we pass "/bin/sh" (the
location of the default shell), then exec will try to transform the current
process into a shell. As an attacker, shells are great because they provide a
convenient and powerful interface to the compromised machine; however, in other
circumstances we may want our shellcode to do different (or additional) things
as well. 
 

We have to understand the syscall **calling convention** when writing our shell
code.  On 32-bit x86 linux, this convention requires the arguments to be placed
in registers followed by the use of the `int 0x80` instruction.  For
information on syscall arguments, check out the following
[x86](https://syscalls.kernelgrok.com/) and
[x86-64](https://filippo.io/linux-syscall-table/) reference guides.

 
**Note:** when a programmer uses `execve` in a C program, they are actually
using a wrapper in the C library to invoke the `sys_execve` **system call**.
In other words, the programmer invokes the library function which in turn
invokes the system call.  In our shellcode, we are going to call `exec`
directly rather than use the library call.


#### Shellcode and "Smashing the Stack"

Let's take a look at the final shellcode given in "Smashing the Stack" (shown
in `shellcodeasm2.c`) and determine the purpose of each instruction.

```ASM
jmp 0x1f                    
popl %esi                   
movl %esi, 0x8(%esi)
xorl %eax,%eax
movb %eax, 0x7(%esi)
movl %eax, 0xc(%esi)
movb $0xb,%al
movl %esi,%ebx
leal 0x8(%esi),%ecx
leal 0xc(%esi),%edx
int $0x80
xorl %ebx,%ebx
movl %ebx,%eax
inc %eax
int $0x80
call -0x24
.string \"/bin/sh\"
```

The ultimate goal is to execute the exec syscall with the appropriate
arguments. Thus, our code is all about setting up these arguments before
eventually executing `int $0x80`. Of course there are a few complications:
 - We have to pass the address of the /bin/sh string, but we have no idea where
   that will be at runtime.
 - We have to use null bytes in our arguments, but we can't include null bytes
   in our shellcode. Similarly, the instructions we use can't contain null
bytes.



##### Breakdown of Individual Instructions

`jmp 0x1f`: Jump to the call instruction at the end. This jump is relative.
Useful because we don't know the absolute address of the this code on the
stack. 

`call -0x24`: We use this instruction to place the address of the `/bin/sh`
string on the stack. How? Remember that call will push the address of the next
instruction to be executed on to the stack before branching, in this case that
would be the address of the /bin/sh string (it might help to think of that
string as masquerading as an instruction).

`.string \"/bin/sh\"`: this is just the bytes for the /bin/sh string. Fun fact:
if we branched here, the CPU would try to execute these bytes as if they were
an instruction.
                    
`popl %esi`: The call instruction transfers control to here. This instruction
loads the address of the `/bin/sh` string in `%esi`.                     

`movl %esi, 0x8(%esi)`: exec requires an array of argument strings. We are
going to start setting up that argument array with the address of the /bin/sh
string as the first element in that array. The `0x8(%esi)` operand refers to:
the address in esi plus 8 bytes. We put this array  at eight bytes past the
address of the /bin/sh string because we need seven bytes for the string and
one byte for the null terminator (which we are about to add). 

`xorl %eax,%eax`: Here we make eax zero so we can use this register for null
bytes later. We need to include a null word (4 null bytes) in our argument
array, but we can't store null bytes in our shellcode. Why? Because many
vulnerable functions, e.g., gets() stop reading strings as soon a null byte is
encountered. In other words, we wouldn't be able to load our entire shellcode
payload if it contained any null bytes.

`movb %eax, 0x7(%esi)`: Move a single null byte to act as a null terminator for
our /bin/sh string. Notice the `b` suffix on `movb`. At this point, the first
argument of our argument array is setup in memory.  

`movl %eax, 0xc(%esi)`: Our knowledge of execve (read the MAN page) tells us
that the last element of the argument array must be a null pointer, so we take
care of that now.

`movb $0xb,%al`: By syscall convention, register eax holds the index (in the
syscall table) of the desired syscall. This let's the hardware/OS know which
syscall to execute. Why do we use a `movb` instruction here instead of `movl
$0xb, %rax`? Answer: the latter instruction includes a 0x0 byte. 
 
`movl %esi,%ebx`: Register ebx holds the first argument to the syscall, i.e.,
the address of the /bin/sh string.

`leal 0x8(%esi),%ecx`: The second argument to the syscall is the address of the
argument array. Recall that the argument array includes: element 1: address of
the /bin/sh string; element 2: null pointer. 

`leal 0xc(%esi),%edx`: The third argument to the syscall is NULL (because we
don't need it here). Rather than add other null byte to the stack, we will just
reuse the second element of the arg-array. In other words, that NULL word is
serving double duty.

`int $0x80`: This instruction triggers the transfer into kernel mode, i.e., it
causes the syscall to be executed.

`xorl %ebx,%ebx ... int $0x80`: These instructs set up the arguments for the
exit syscall. Note: this code will only execute if exec fails.

```
View of Memory

        esi+0x7
           |
           |
esi+0x0    |   esi+0x0C
   |       |      |
   v       v      v
   |/bin/sh|0|ADDR|NULL|
             ^
             |
          esi+0x08

```


#### Self-modifying Code??

"Smashing the Stack" makes a strange comment about the code "self-modifying".
This statement is misleading because the shellcode itself isn't modified, just
the bytes immediately after that shellcode. This isn't an issue when the
shellcode is on the stack---as it will be when we perform the attack. 

However, when are testing the shellcode using the `___asm___` capability (as
shown in the paper), the shellcode will be put into the same region of memory
as other code and the "memory" modified by the shellcode will fall in the page
as the shellcode.  Because of paging---specifically, all memory in the same
page has the same permissions---this memory will also be marked as read-only.

### Testing your Shellcode

The code given in "Smashing the Stack" for testing the shell code is a bit
ugly. Here's what they do (which involves overwriting the saved return address
on the stack:

```C
char shellcode[] = 
  "\xeb\x1f\x5e\x89\x76\x08\x31\xc0\x88\x46\x07\x89\x46\x0c\xb0\x0b" 
  "\x89\xf3\x8d\x4e\x08\x8d\x56\x0c\xcd\x80\x31\xdb\x89\xd8\x40\xcd" 
  "\x80\xe8\xdc\xff\xff\xff/bin/sh";

void main() { 
  int *ret;
  ret = (int *)&ret + 2; 
  (*ret) = (int)shellcode;
}
```

It is much easier to use a function pointer: 

```C
void main() { 
  void (*s)() = (void *)shellcode;
  s();
}
```

### Misc. Tips


**32-bit vs. 64-bit shell code.** Be careful that you don't use 32-bit
shellcode on a 64-bit system. A good indicator is the use of `int 0x80` versus
`syscall`, the former is used primarily on 32-bit systems and the latter is
used in 64-bit.  Note, you can use `int 0x80` in a 64-bit system but it can be
tricky to make sure that the pointer sizes are only 4 bytes (e.g., pointers to
the stack typically require more than 4 bytes). 


**Why does the popped shell exit immediately?** Sometimes when you pop a shell,
it appears like nothing happened even though your shellcode seemed to execute.
This could occur because stdin is closed, causing the shell to exit
immediately. To fix this problem, you either get some fancy shell code (not
really that fancy) to reopen stdin, or you use the cat trick: 

```bash
(python exploit.py; cat) | ./vulnerable_prog 
```

**How to check if the stack is executable:** 
 - `objdump -x foo`
 - Look in the "Program Header" section of the output.
 - Check the permission bits for the stack, e.g., "flags rw-"
 - "flags rw-" means the stack is not executable.

Alternatively, if you need to check if the **heap is executable**, you have to take
a more roundabout approach:
 - run `info proc map` inside of gdb to get the address of the heap
 - run the program outside of gdb and suspend it
 - get the pid of the suspended process using `ps -a`
 - get the memory map of the process using `pmap PID`
 - match the addresses (assuming no aslr), the heap might be labeled `anon` in
   `pmap`.

**Debugging tips:**
 - Use the `int3` instruction to debug shellcode. It's simple to use, just throw
in the opcode 0xCC. The following is a good starter: `print '\x90'*100 + '\xCC'*4`.
 - If you are getting an `errno` of `-38` (i.e., function not implemented) then
   make sure that you have the right argument in `rax` when you issue the
`syscall`. 
 - The return value of a system call will be placed in the `rax` register. If
   the return is negative, an error occurred; subtract the return value from
zero and look up the number with `errno -l`.  
