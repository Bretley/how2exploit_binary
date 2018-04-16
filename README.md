# how2exploit_binary: get your hack on.

### A note from the creator

Greetings, fellow hacker, hobbyist, or computer enthusiast. If you've been
looking for a place to start learning binary exploitation, then you're in luck.
This tutorial is intended for anyone with experience in coding, ideally C or
C++, but I only knew Python when I started.

Written by someone who is just barely better than "incompetent," I'll be
explaining how I learned my skills. These tutorials will be a bit long winded,
but hopefully they will be informative and entertaining. Please feel free to
contact me about any clarifications that should be included in the tutorials.

**This is intended for Linux. It's free if you don't already have it. Don't
want to dual boot? Get a VM.**

-Best of luck

[Bretley](https://github.com/Bretley)

## The Grand Glossary of Terms

I've compiled this list of as many useful things as I could find. It contains
all sorts of goodies that I wish I had found or had explained to me earlier. If
you have a question, it can probably be answered in here. Otherwise, get your
Google-Fu on

* [The Glossary](terms)

## External Tools.

I strongly recommend you install and use the following tools to make your life
a bit easier:

* [longld/peda](https://github.com/longld/peda/): I use this tool in all of
  these tutorials. It provides a wide range of useful functions and makes `gdb`
  far more user friendly. Just follow the installation instructions in the repo.

* [Gallopsled/pwntools](https://github.com/Gallopsled/pwntools): pwntools is an
  exploit framework built in my favorite language, python. It has a whole slew
  of useful functions and chicanery that makes the exploit process more fun and
  less painful. Install with: `$ sudo pip install pwntools`

## Introductory Tutorials:

* [Setup Script](./install.sh)
* [Intro 1: What is a binary, really?](intro-1)
    * [Companion Video](https://youtu.be/6cNbKnxbAWw)
    * [Areece x86 Calling Conventions](http://codearcana.com/posts/2013/05/21/a-brief-introduction-to-x86-calling-conventions.html)
* [Intro 2: Screwing around with the stack](intro-2)

## Buffer Overflows and ROP:

* [1:   The power of SEGFAULT](exercise-1)
* [2:   Build your own `system()`](exercise-2)
* [3:   Follow the Yellow Brick Functions](exercise-3)
* [3.5: Learning pwntools](exercise-3.5)
* [4:   Pay a Visit to Your Local Library](exercise-4)

## Heap Exploitation:

* More to come here soon ;)
