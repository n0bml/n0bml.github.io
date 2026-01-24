---
layout: post
title: Raspberry Pi Pico 2 ARM Development Environment Setup
date: 2026-01-23
tags: pico2 arm
---

I've always been interested in low level embedded development.  When I'm programming the closer to the metal I can get the more I enjoy the project.  I was digging through my [junk box](https://en.wikipedia.org/wiki/Junk_box) for some resistors and found a few Pico [1](https://www.raspberrypi.com/products/raspberry-pi-pico/)s and [2](https://www.raspberrypi.com/products/raspberry-pi-pico-2/)s from [Raspberry Pi](https://www.raspberrypi.com/) gathering dust.  What else could I do but dig them out and write some code to run on them.

As you might know the `Pico 1` has the [RP2040](https://www.raspberrypi.com/products/rp2040/) microcontroller which contains a dual core [ARM](https://www.arm.com/) [Cortex-M0+](https://developer.arm.com/Processors/Cortex-M0-Plus) CPU.  The `Pico 2` contains the newer [RP2350](https://www.raspberrypi.com/products/rp2350/) microcontroller which contains both a dual core [ARM](https://www.arm.com/) [Cortex-M33](https://developer.arm.com/Processors/Cortex-M33) and a dual core [Hazard3](https://github.com/Wren6991/Hazard3) [RISC-V](https://riscv.org/) CPU.  Both of these boards have good support in the Raspberry Pi ecosystem and can be programmed with C/C++, Python, or even assembly.

The [Pico series documentation](https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html) page has good instructions for setting up the SDK the compiler for the ARM CPU but doesn't include any information about the compiler for the RISC-V CPU.  This guide about how to setup the tool chains and is the first part of a series about creating one [UF2](https://github.com/microsoft/uf2) binary that will run on the RP2040 ARM CPU, and both the RP2350 ARM CPU and RISC-V CPU.

# Setup the Development Environment.

The easiest way to setup the Pico SDK, and associated tools is to download and run[^1] the [pico_sdk.sh](https://github.com/raspberrypi/pico-setup/blob/master/pico_setup.sh) script from the Raspberry Pi [pico-setup](https://github.com/raspberrypi/pico-setup) GitHub project.  This script will install the GCC ARM compiler, [SDK](https://github.com/raspberrypi/pico-sdk), [examples](https://github.com/raspberrypi/pico-examples), [extras](https://github.com/raspberrypi/pico-extras), [playground](https://github.com/raspberrypi/pico-playground), including the [picotool](https://github.com/raspberrypi/picotool), and [debugprobe](https://github.com/raspberrypi/debugprobe) utilities.  The setup script works on [Debian](https://www.debian.org/) based distributions.  If you are running a different flavor of Linux you can still use the `pico_sdk.sh` script but you will have to run the appropriate commands manually.  This is an exercise left to the reader.

# Testing the Compiler.

To test the compilers we will compile a small C snippet and emit the generated instructions for the two cross compilers.  The sample we will use some code from the book [From Mathematics to Generic Programming](https://www.fm2gp.com/) by [Alexander Stepanov](https://en.wikipedia.org/wiki/Alexander_Stepanov) and [Daniel E. Rose](https://www.ordinology.com/).  The snippet we will use is:

## Sample C Code (multiply.c).

```c
#include <stdbool.h>

bool odd(int n) { return n & 0x01; }
int half(int n) { return n >> 1; }

static int mult_acc(int r, int n, int a) {
    while (1) {
        if (odd(n)) {
  	        r += a;
  	        if (n == 1) return r;
  	    }
  	    n = half(n);
  	    a += a;
    }
}

int multiply(int n, int a) {
    while (!odd(n)) {
        a += a;
        n = half(n);
    }
    if (n == 1) return a;
    return mult_acc(a, half(n - 1), a + a);
}
```

## Compile and Generate ARM Assembly.

Using this `Makefile` we can compile the sample to confirm the ARM compiler is working.  I find using [GNU Make](https://www.gnu.org/software/make/) easier than typing in all the commands by hand and less error prone.

```
CC=arm-none-eabi-gcc
CPPLAGS+=-Wall -Wextra -Wshadow -Wnon-virtual-dtor -Wuseless-cast -Wpedantic -pedantic
CFLAGS+=-std=c17 -g -O2

multiply.o: multiply.c
	$(CC) $(CPPFLAGS) $(CFLAGS) -o $@ $<

clean:
	$(RM) multiply.o multiply.s
```

## Running Make.

This is the output I get when I ran `make` on my develoment box, as you can see there were no errors or warning generated.  So the compiler is working as expected at the moment.  The

```
$ make
arm-none-eabi-gcc  -std=c17 -g -O2 -c -o multiply.o multiply.c
```

## Examining the Generated Assembly.

Using `objdump` we can view the generated ARM assembly for the sample code.  There is no output generated for the `mult_acc` function as it is marked as `static inline`, so the compiler rolls its implementation into the `multiply` function.  I did not use `static inline` with the `odd` and `half` functions as they are used by other functions that aren't shown in this sample.

```
arm-none-eabi-objdump -d multiply.o
```

### `odd` Function.

```
00000000 <odd>:
   0:   e2000001        and     r0, r0, #1
   4:   e12fff1e        bx      lr
```

### `half` Function.

```
00000008 <half>:
   8:   e1a000c0        asr     r0, r0, #1
   c:   e12fff1e        bx      lr
```

### `multiply` Function.

```
00000010 <multiply>:
  10:   e3100001        tst     r0, #1
  14:   e1a03000        mov     r3, r0
  18:   1a000003        bne     2c <multiply+0x1c>
  1c:   e1a030c3        asr     r3, r3, #1
  20:   e3130001        tst     r3, #1
  24:   e1a01081        lsl     r1, r1, #1
  28:   0afffffb        beq     1c <multiply+0xc>
  2c:   e3530001        cmp     r3, #1
  30:   e1a00001        mov     r0, r1
  34:   012fff1e        bxeq    lr
  38:   e2433001        sub     r3, r3, #1
  3c:   e1a030c3        asr     r3, r3, #1
  40:   e1a02081        lsl     r2, r1, #1
  44:   e3130001        tst     r3, #1
  48:   0a000002        beq     58 <multiply+0x48>
  4c:   e3530001        cmp     r3, #1
  50:   e0800002        add     r0, r0, r2
  54:   012fff1e        bxeq    lr
  58:   e1a030c3        asr     r3, r3, #1
  5c:   e1a02082        lsl     r2, r2, #1
  60:   eafffff7        b       44 <multiply+0x34>
```

# Sample Files.

The sample files mentioned in this post can be viewed on [GitHub](https://github.com/n0bml/blog-samples/tree/main/pico2-arm-dev-setup).

# Next Steps.

Future posts in this series will show:

1. Create a [CMake](https://cmake.org/) project to generate executable binary and load it on to a `Pico 2` board.
1. Update the `CMake` project to generate a single binary and load it on `Pico` and `Pico 2` boards.
1. Setup the RISC-V Hazard3 compiler toolchain and test the compiler with the sample code.
1. Create a single [UF2](https://github.com/microsoft/uf2) binary that supports ARM on the `Pico` board, ARM on the `Pico 2` board and RISC-V on the `Pico 2` board.

# Pico 2 ARM and RISC-V Development Series Links.

1. Raspberry Pi Pico 2 ARM Development Environment Setup (this post)
1. Raspberry Pi Pico 2 RISC-V Development Environment Setup.  (TBD)
1. Raspberry Pi Pico 2 Basic CMake Project.  (TBD)
1. Raspberry Pi Pico 2 Extend Project for Pico 1 and Pico 2.  (TBD)
1. Raspberry Pi Pico 2 Universal UF2 Project.  (TBD)

# Footnotes.

[^1]: Never run a script you download without reviewing the source.  Supply chain attacks exist, yo.
