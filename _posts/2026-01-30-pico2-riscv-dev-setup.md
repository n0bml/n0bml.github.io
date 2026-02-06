---
layout: post
title: Raspberry Pi Pico 2 RISC-V Development Environment Setup
date: 2026-01-30
tags: pico2 riscv
---

In the [first post](https://n0bml.github.io/pico2-arm-dev-setup/) of this series I wrote about how to setup the development environment for the [Raspberry Pi](https://www.raspberrypi.com/) [Pico 2](https://www.raspberrypi.com/products/raspberry-pi-pico-2/) microcontroller for the ARM CPU.  In this post I am going to cover how to setup the development environment for the [RISC-V Hazard3](https://github.com/Wren6991/Hazard3) CPU.  Unlike the ARM development environment there isn't a setup script hosted on the Raspberry Pi GitHub pages.  Luckily the Raspberry Pi [forums](https://forums.raspberrypi.com/) are very active and people there guided me to the information I needed.

# Compiling the RISC-V and Hazard3 Toolchains.

In the [Raspberry Pi Pico C/C++ SDK](https://datasheets.raspberrypi.com/pico/raspberry-pi-pico-c-sdk.pdf) document section 2.10, titled "Supporting both RP2040 and RP2350," has the important steps we need.  I've changed the directions slightly to match my preferences and provided shell scripts in the [sample files](https://github.com/n0bml/blog-samples/tree/main/pico2-riscv-dev-setup) included with this post, so you don't have to copy an paste the text from this post.

**Note:** The Hazard3 is a RISC-V CPU but for this post I'll refer to the compiler toolchain built from the steps in the C/C++ SDK as *RISC-V* and the steps from the Hazard3 repository as *Hazard3*.

## Prerequisites

If, like me, you compile software from source you probably have all of these packages installed on your system.  Just in case here are the ones you need to install:

```shell
sudo apt-get install autoconf automake autotools-dev curl python3 \
    python3-pip libmpc-dev libmpfr-dev libgmp-dev gawk \
    build-essential bison flex texinfo gperf libtool patchutils \
    bc zlib1g-dev libexpat-dev ninja-build git cmake \
    libglib2.0-dev libslirp-dev
```

## Compiling the RISC-V Toolchain.

First, compile and install the standard [RISC-V GNU toolchain](https://github.com/riscv/riscv-gnu-toolchain) as documented in the C/C++ SDK.  On my system, with a 13th Gen Intel Core i5-13400F with 10 cores, `make` takes about 15 minutes to build the compiler.  It also took around 25 minutes to clone the `gnu-14` branch from the `gcc-mirror` repository.  Making the entire script take around 40 minutes to finish.

```shell
PICO_PATH="${HOME}/pico"
TOOLCHAIN_PATH="${PICO_PATH}/toolchains"

git clone https://github.com/riscv/riscv-gnu-toolchain
cd riscv-gnu-toolchain

git clone https://github.com/gcc-mirror/gcc gcc-14 -b releases/gcc-14

./configure --prefix="${TOOLCHAIN_PATH}/gcc14-rp2350-no-zcmp" \
    --with-arch=rv32ima_zicsr_zifencei_zba_zbb_zbs_zbkb_zca_zcb \
    --with-abi=ilp32 \
    --with-multilib-generator="rv32ima_zicsr_zifencei_zba_zbb_zbs_zbkb_zca_zcb-ilp32--;rv32imac_zicsr_zifencei_zba_zbb_zbs_zbkb-ilp32--" \
    --with-gcc-src="$(pwd)/gcc-14"

mkdir -p "${TOOLCHAIN_PATH}/gcc14-rp2350-no-zcmp"
time make -j "$(nproc)"
```

# Compiling the Hazard3 Toolchain.

The RISC-V CPU in the RP2350 microcontroller is a [Hazard3](https://github.com/Wren6991/Hazard3).  The GitHub repository for the Hazard3 has it's own set of build instructions.  I've also built this toolchain to compare the two.  Being a curious sort I wondered if the generated instructions would be different between the two toolchains.  To build it we follow these set of instructions, which are included in the `hazard3-setup.sh` file in the [sample files](https://github.com/n0bml/blog-samples/tree/main/pico2-riscv-dev-setup) repository.

``` shell
PICO_PATH="${HOME}/pico"
TOOLCHAIN_PATH="${PICO_PATH}/toolchains"

git clone https://github.com/riscv/riscv-gnu-toolchain gcc14-hazard3-rp2350
cd gcc14-hazard3-rp2350

git clone https://github.com/gcc-mirror/gcc gcc-14 -b releases/gcc-14

./configure --prefix="${TOOLCHAIN_PATH}/gcc14-hazard3-rp2350" --with-arch=rv32ia_zicsr_zifencei --with-abi=ilp32 --with-multilib-generator="rv32i_zicsr_zifencei-ilp32--;rv32im_zicsr_zifencei-ilp32--;rv32ia_zicsr_zifencei-ilp32--;rv32ima_zicsr_zifencei-ilp32--;rv32ic_zicsr_zifencei-ilp32--;rv32imc_zicsr_zifencei-ilp32--;rv32iac_zicsr_zifencei-ilp32--;rv32imac_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zicsr_zifencei-ilp32--;rv32imc_zba_zbb_zbs_zicsr_zifencei-ilp32--;rv32imac_zba_zbb_zbs_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zbkb_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zbkb_zicsr_zifencei-ilp32--;rv32imc_zba_zbb_zbs_zbkb_zicsr_zifencei-ilp32--;rv32imac_zba_zbb_zbs_zbkb_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbc_zbs_zbkb_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbc_zbs_zbkb_zicsr_zifencei-ilp32--;rv32imc_zba_zbb_zbc_zbs_zbkb_zicsr_zifencei-ilp32--;rv32imac_zba_zbb_zbc_zbs_zbkb_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zbkb_zbkx_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zbkb_zbkx_zicsr_zifencei-ilp32--;rv32imc_zba_zbb_zbs_zbkb_zbkx_zicsr_zifencei-ilp32--;rv32imac_zba_zbb_zbs_zbkb_zbkx_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbc_zbs_zbkb_zbkx_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbc_zbs_zbkb_zbkx_zicsr_zifencei-ilp32--;rv32imc_zba_zbb_zbc_zbs_zbkb_zbkx_zicsr_zifencei-ilp32--;rv32imac_zba_zbb_zbc_zbs_zbkb_zbkx_zicsr_zifencei-ilp32--;rv32i_zca_zicsr_zifencei-ilp32--;rv32im_zca_zicsr_zifencei-ilp32--;rv32ia_zca_zicsr_zifencei-ilp32--;rv32ima_zca_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zca_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zca_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zbkb_zca_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zbkb_zca_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbc_zbs_zbkb_zca_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbc_zbs_zbkb_zca_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zbkb_zbkx_zca_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zbkb_zbkx_zca_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbc_zbs_zbkb_zbkx_zca_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbc_zbs_zbkb_zbkx_zca_zicsr_zifencei-ilp32--;rv32i_zca_zcb_zicsr_zifencei-ilp32--;rv32im_zca_zcb_zicsr_zifencei-ilp32--;rv32ia_zca_zcb_zicsr_zifencei-ilp32--;rv32ima_zca_zcb_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zca_zcb_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zca_zcb_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zbkb_zca_zcb_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zbkb_zca_zcb_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbc_zbs_zbkb_zca_zcb_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbc_zbs_zbkb_zca_zcb_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zbkb_zbkx_zca_zcb_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zbkb_zbkx_zca_zcb_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbc_zbs_zbkb_zbkx_zca_zcb_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbc_zbs_zbkb_zbkx_zca_zcb_zicsr_zifencei-ilp32--;rv32i_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32im_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32ia_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32ima_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zbkb_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zbkb_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbc_zbs_zbkb_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbc_zbs_zbkb_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbs_zbkb_zbkx_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbs_zbkb_zbkx_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32im_zba_zbb_zbc_zbs_zbkb_zbkx_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32ima_zba_zbb_zbc_zbs_zbkb_zbkx_zca_zcb_zcmp_zicsr_zifencei-ilp32--;rv32i_zmmul_zicsr_zifencei-ilp32--;rv32ia_zmmul_zicsr_zifencei-ilp32--;rv32ic_zmmul_zicsr_zifencei-ilp32--;rv32iac_zmmul_zicsr_zifencei-ilp32--;rv32i_zca_zmmul_zicsr_zifencei-ilp32--;rv32ia_zca_zmmul_zicsr_zifencei-ilp32--;rv32i_zca_zcb_zmmul_zicsr_zifencei-ilp32--;rv32ia_zca_zcb_zmmul_zicsr_zifencei-ilp32--;rv32i_zca_zcb_zcmp_zmmul_zicsr_zifencei-ilp32--;rv32ia_zca_zcb_zcmp_zmmul_zicsr_zifencei-ilp32--;rv32e_zicsr_zifencei-ilp32e--;rv32ema_zicsr_zifencei-ilp32e--;rv32emac_zicsr_zifencei-ilp32e--;rv32ema_zicsr_zifencei_zba_zbb_zbc_zbkb_zbkx_zbs_zca_zcb_zcmp-ilp32e--" --with-gcc-src="$(pwd)/gcc-14"

mkdir -p "${TOOLCHAIN_PATH}/gcc14-hazard3-rp2350"
time make -j "$(nproc)"
```

Go and have lunch, watch a show, or something.  On my system `make` takes over 90 minutes to finish.  Also, cloning the `gcc-14` branch of the GCC repository takes about 25 minutes.  If you're going to build both versions of the compilers it would save you some time to clone the repository once and update the scripts to point to the common directory.  Just remember to run `make distclean` between building the two compilers to start with a clean slate.

# Testing the Compilers.

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

## Compile the Sample Code.

Using this `Makefile` we can compile the sample to confirm the two RISC-V compilers are working.

```make
TOOLCHAINS=$(HOME)/pico/toolchains

RISCVCC=$(TOOLCHAINS)/gcc14-rp2350-no-zcmp/bin/riscv32-unknown-elf-gcc
HAZARD3CC=$(TOOLCHAINS)/gcc14-hazard3-rp2350/bin/riscv32-unknown-elf-gcc

CPPFLAGS+=-Wall -Wextra -Wshadow -Wuseless-cast -Wpedantic -pedantic
CFLAGS+=-std=c17 -g -O2

all: riscv-multiply.o hazard3-multiply.o

riscv-multiply.o: multiply.c
	$(RISCVCC) $(CPPFLAGS) $(CFLAGS) -c -o $@ $<

hazard3-multiply.o: multiply.c
	$(HAZARD3CC) $(CPPFLAGS) $(CFLAGS) -c -o $@ $<

clean:
	$(RM) riscv-multiply.o hazard3-multiply.o
```

Running make to build the same sample code with the RISC-V and Hazard3 compilers did not generate any errors or warnings.  I count this as a success for at this stage in the project.

# Reviewing Generated RISC-V Assembly.

Using `objdump` we are able to review the generated assembly instructions for both of the compilers.

## RISC-V Generated Assembly.

```shell
$ ~/pico/toolchains/gcc14-rp2350-no-zcmp/bin/riscv32-unknown-elf-objdump -d riscv-multiply.o
00000000 <odd>:
   0:   8905                    andi    a0,a0,1
   2:   8082                    ret

00000004 <half>:
   4:   8505                    srai    a0,a0,0x1
   6:   8082                    ret

00000008 <multiply>:
   8:   00157713                andi    a4,a0,1
   c:   87aa                    mv      a5,a0
   e:   e711                    bnez    a4,1a <.L5>

00000010 <.L6>:
  10:   8785                    srai    a5,a5,0x1
  12:   0017f713                andi    a4,a5,1
  16:   0586                    slli    a1,a1,0x1
  18:   df65                    beqz    a4,10 <.L6>

0000001a <.L5>:
  1a:   4605                    li      a2,1
  1c:   852e                    mv      a0,a1
  1e:   00c78f63                beq     a5,a2,3c <.L4>
  22:   17fd                    addi    a5,a5,-1
  24:   8785                    srai    a5,a5,0x1
  26:   00159713                slli    a4,a1,0x1
  2a:   a019                    j       30 <.L9>

0000002c <.L8>:
  2c:   8785                    srai    a5,a5,0x1
  2e:   0706                    slli    a4,a4,0x1

00000030 <.L9>:
  30:   0017f693                andi    a3,a5,1
  34:   dee5                    beqz    a3,2c <.L8>
  36:   953a                    add     a0,a0,a4
  38:   fec79ae3                bne     a5,a2,2c <.L8>

0000003c <.L4>:
  3c:   8082                    ret
```

## Hazard3 Generated Assembly.

``` shell
$ ~/pico/toolchains/gcc14-hazard3-rp2350/bin/riscv32-unknown-elf-objdump -d hazard3-multiply.o
Disassembly of section .text:

00000000 <odd>:
   0:   00157513                andi    a0,a0,1
   4:   00008067                ret

00000008 <half>:
   8:   40155513                srai    a0,a0,0x1
   c:   00008067                ret

00000010 <multiply>:
  10:   00157713                andi    a4,a0,1
  14:   00050793                mv      a5,a0
  18:   00071a63                bnez    a4,2c <.L5>

0000001c <.L6>:
  1c:   4017d793                srai    a5,a5,0x1
  20:   0017f713                andi    a4,a5,1
  24:   00159593                slli    a1,a1,0x1
  28:   fe070ae3                beqz    a4,1c <.L6>

0000002c <.L5>:
  2c:   00100613                li      a2,1
  30:   00058513                mv      a0,a1
  34:   02c78663                beq     a5,a2,60 <.L4>
  38:   fff78793                addi    a5,a5,-1
  3c:   4017d793                srai    a5,a5,0x1
  40:   00159713                slli    a4,a1,0x1
  44:   00c0006f                j       50 <.L9>

00000048 <.L8>:
  48:   4017d793                srai    a5,a5,0x1
  4c:   00171713                slli    a4,a4,0x1

00000050 <.L9>:
  50:   0017f693                andi    a3,a5,1
  54:   fe068ae3                beqz    a3,48 <.L8>
  58:   00e50533                add     a0,a0,a4
  5c:   fec796e3                bne     a5,a2,48 <.L8>

00000060 <.L4>:
  60:   00008067                ret
```

# Final Thoughts.

For this simple example I don't see any difference in the code generated by the two compilers.  I am going to keep both of them installed on my system and continue to compare the generated code.  If you just want to get started compiling code for the RISC-V CPU on the Pico 2 I would follow the instructions in the C/C++ SDK document.  Being written by people developing the microcontroller and software it should "just work."

# Sample Files.

The sample files mentioned in this post can be viewed on [GitHub](https://github.com/n0bml/blog-samples/tree/main/pico2-riscv-dev-setup).

# Next Steps.

In the next post in this series we will create a [CMake](https://cmake.org/) project to generate executable binary and load it on to a `Pico 2` board.

# Pico 2 ARM and RISC-V Development Series Links.

1. [Raspberry Pi Pico 2 ARM Development Environment Setup](https://n0bml.github.io/pico2-arm-dev-setup/).
1. Raspberry Pi Pico 2 RISC-V Development Environment Setup (this post).
1. [Raspberry Pi Pico 2 Basic CMake Project](https://n0bml.github.io/pico2-basic-cmake-project/).
1. Raspberry Pi Pico 2 Extend Project for Pico 1 and Pico 2 (TBD).
1. Raspberry Pi Pico 2 Extend Project for Pico 2 RISC-V CPU (TBD).
1. Raspberry Pi Pico 2 Universal UF2 Project (TBD).
