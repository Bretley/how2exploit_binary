# pwntools Overview

**Documentation: https://pwntools.readthedocs.io**

First things first:

```python
from pwn import *
```

That's just a generic import statement.

```python
context(arch='i386', os='linux')
```

This just sets the context for other functions that we'll describe later.

```python
binary = ELF("some_challenge")
libc = ELF("some_libc")
```

This part adds two ELF objects, binary and libc. ELF objects are supremely useful -- they give you access to a wide array of methods and data fields. I almost always have both of these lines in my script, even if the libc one is commented out.

```python
r = process("./some_challenge")
```

This simply executes the challenge (in the same directory.)

Alternatively:

```python
r = remote("127.0.0.1",1337) #<-- Replace with actual HOST,PORT
```

will run it remotely (many CTFs will not give you a full shell, just a host and
a port to connect to the binary.)

Many of you will remember taking adresses and turning them into python
escape sequences by hand.

If the address of the `write()` function is `0xdeadbeef`, the escaped address for `write()` would be `\xef\xbe\xad\xde`.

`pwntools` can take care of this for us!

```python
write = p32(binary.symbols["write"])
```

This "packs" (converts to the escape seqence, sort of) the address of `write()` for us on a 32 bit machine. `p64()` also exists, for 64 bit machines. Another thing to be cognizant of is the difference between Big and Little Endian memory encoding. Make sure you know what format the system you're writing an exploit for is using.

Assuming `r` is an instantiated process or remote, you can now use these methods to communicate with the binary.

```python
r.sendline("This sends a string with a newline appended to the end")
r.send("This also sends a string")
```

Reading this information is one thing. Getting real experience is another.
**At this point I would strongly recommend solving the first 3 challenges using pwntools.**
