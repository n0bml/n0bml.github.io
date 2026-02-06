---
layout: post
title: Raspberry Pi Pico 2 Basic CMake Project
date: 2026-02-06
tags: pico2 riscv cmake
---

In my [last post](https://n0bml.github.io/pico2-riscv-dev-setup/) I covered how to setup the [RISC-V](https://riscv.org/) toolchain for the [Hazard3](https://github.com/Wren6991/Hazard3) CPU in the Pico 2.  In this post I will set up a simple project using [CMake](https://cmake.org/) to blink the on-board LED.  Be sure you have installed the Pico SDK using [part 1](https://n0bml.github.io/pico2-arm-dev-setup/) of this series.  Now, let's get into the details.

# Quick Start

The C/C++ SDK documentation has a [quick start](https://www.raspberrypi.com/documentation/microcontrollers/c_sdk.html#quick-start-your-own-project) guide to creating your own project.  I used those instructions as a starting point but I've added a few extras to make life easier building and testing programs for the Pico 2.  You can see the full example in the [sample project](https://github.com/n0bml/blog-samples/tree/main/pico2-basic-cmake-project) that goes with this post.

# CMake Configuration

For my personal projects I prefer using [GNU Make](https://www.gnu.org/software/make/) but the SDK provided by Raspberry Pi uses [CMake](https://cmake.org/), so I will also use it.  It does simplify specifying the particular microcontroller board and features.  If you are not familiar with `CMake`, there is a [tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html) on their website that will get you started.  Here's a breakdown of the various sections of the [CMakeLists.txt](https://github.com/n0bml/blog-samples/blob/main/pico2-basic-cmake-project/CMakeLists.txt) file:

## Standard Header.

We start by specifying the minimum version of `CMake` that is required.  I also always enable the `CMAKE_EXPORT_COMPILE_COMMANDS` feature, which causes `CMake` to generate the file `compile_commands.json` in the build output directory.  This [compilation database](https://clang.llvm.org/docs/JSONCompilationDatabase.html) is used by tools like `clang-tidy` and [language servers](https://microsoft.github.io/language-server-protocol/) to provide intelligent editor features.

``` cmake
cmake_minimum_required(VERSION 3.13)

# Always export compile commands for tools like eglot and clang-tidy.
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

## Default to Debug Build.

If the project is being built from a local [Git](https://git-scm.com/) repository, I prefer a debug build by default, otherwise a release with debug info is created.  This section checks for `.git/` directory in the source directory and sets the default build type to "Debug."  This can be overridden by specifying the build type on the `cmake` command line.

``` cmake
set(default_build_type "RelWithDebInfo")
if(EXISTS "${CMAKE_SOURCE_DIR}/.git")
  set(default_build_type "Debug")
endif()

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPE)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE STRING "Choose type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()
```

## Default to Pico 2 Microcontroller.

Most of the microcontrollers I use are Pico 2 boards, so I set that as the default board type, unless it has been specified on the `CMake` command line.

``` cmake
if(NOT PICO_BOARD)
  set(PICO_BOARD "pico2")
  message(STATUS "Setting board type to '${PICO_BOARD}' as none was specified.")
endif()
```

## Ensure the PICO_SDK_PATH Environment Variable is Defined.

If you setup the Pico SDK using the [pico_setup.sh](https://github.com/raspberrypi/pico-setup) script then a few environment variables were set, such as:

- `PICO_SDK_PATH`
- `PICO_EXTRAS_PATH`
- `PICO_EXAMPLES_PATH`
- `PICO_PLAYGROUND_PATH`

We need the `PICO_SDK_PATH` environment variable to be able to find the appropriate headers, libraries and extra files, and we verify the variable exists with:

``` cmake
# Include the Pico SDK CMake support.  This MUST happen before project().
if(NOT DEFINED ENV{PICO_SDK_PATH} OR "$ENV{PICO_SDK_PATH}" STREQUAL "")
  message(FATAL_ERROR "Environment variable PICO_SDK_PATH is missing or empty.")
endif()
include("$ENV{PICO_SDK_PATH}/external/pico_sdk_import.cmake")
```

## Define Project Name, Languages and Initialize the SDK.

All `CMake` projects start with the `project()` command, which sets the name of the project, and optionally other information such as:

- Version.
- Description.
- Homepage URL.
- Programming Languages.

In this example I named the project `pico2_basic_cmake` and specified the programming languages as C, C++, and assembly.  While I only use C in this example, the SDK uses C++ and assembly in different areas, so I always add all three.

``` cmake
project(pico2_basic_cmake C CXX ASM)
pico_sdk_init()
```

## Add Source Files and Specify Compiler Options.

Now for the "meat" of the executable.  The C source files that make up the project.  `CMake` supports more complicated projects than this simple example but for now we only need to add the one source file, [pico2_basic_cmake.c](https://github.com/n0bml/blog-samples/blob/main/pico2-basic-cmake-project/pico2_basic_cmake.c).  I also specify the use of the [C17](https://en.wikipedia.org/wiki/C17_(C_standard_revision)) version of the C standard, enable some extra warnings, and include the standard library from the Pico SDK.

``` cmake
add_executable("${PROJECT_NAME}" pico2_basic_cmake.c)
target_compile_features("${PROJECT_NAME}" PRIVATE c_std_17)
target_compile_options("${PROJECT_NAME}" PRIVATE -Wall -Wextra)
target_link_libraries("${PROJECT_NAME}" pico_stdlib)
```

## Enable Standard I/O on USB.

For this project I want to see the output from functions like `puts` on a USB serial connection, so we need to tell the SDK to enable USB for stdio and disable UART for stdio.

``` cmake
pico_enable_stdio_usb("${PROJECT_NAME}" 1)
pico_enable_stdio_uart("${PROJECT_NAME}" 0)
```

## Create Extra Output Files.

Finally, we instruct the SDK to create extra files when building the project.  Without calling the `pico_add_extra_output` function only a [.elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) file is generated.  The extra files include:

- [.uf2](https://github.com/microsoft/uf2): The standard file for flashing USB devices.
- [.hex](https://en.wikipedia.org/wiki/Intel_HEX): An Intel HEX format file.
- `.bin`: A raw binary file of the executable.
- `.map`: A file that details memory usage.

``` cmake
pico_add_extra_outputs("${PROJECT_NAME}")
```

# Sample C Source Code (`pico2_basic_cmake.c`).

For this post I wrote a [small sample program](https://github.com/n0bml/blog-samples/blob/main/pico2-basic-cmake-project/pico2_basic_cmake.c) that blinks the on-board LED and prints a message to stdout.  The various sections are outlined below.

## Include Headers.

The program doesn't require including a lot of standard headers.  We only have these three and `pico/binary_info.h` is only for convenience when using `picotool` to determine what has been loaded on the Pico 2.

``` c
#include <stdio.h>
#include <pico/stdlib.h>
#include <pico/binary_info.h>
```

## Ensure `PICO_DEFAULT_LED_PIN` Is Defined.

On the Pico 2, the on-board LED is typically attached to pin 25 and the SDK defines the macro `PICO_DEFAULT_LED_PIN`, which we use.  It is possible to use this program with a different microcontroller that uses the RP2350 core, which might use a different pin.  We make sure the macro is defined and has a value.

``` c
#if !defined(PICO_DEFAULT_LED_PIN) && !(PICO_DEFAULT_LED_PIN + 0)
#error "The macro PICO_DEFAULT_LED_PIN must be defined!"
#endif
```

## Set the Delay For Blinking.

By default we use a 1000 millisecond delay for flashing the LED and printing the message.  It is a good practice to allow values like this to be overridden with compiler flags.

``` c
#if !defined(LED_DELAY_MS)
#define LED_DELAY_MS 1000
#endif
```

## Extra `picotool` Information.

`bi_decl` adds information to the binary image that `picotool` can use to display the program name, description and other information.  It isn't required for any program running on the Pico 2 but it is a good idea to include it.  How many times have you picked up a microcontroller from a box and wondered what was installed on it?  I have more than once and this lets me plug it in and query the board.  An example of the output is:

``` text
Program Information
 name:          pico2_basic_cmake
 description:   Pico 2 Basic CMake Project.
 features:      USB stdin / stdout
 binary start:  0x10000000
 binary end:    0x10004e04
 target chip:   RP2350
 image type:    ARM Secure
```

``` c
bi_decl(bi_program_name("pico2_basic_cmake"));
bi_decl(bi_program_description("Pico 2 Basic CMake Project."));
bi_decl(bi_1pin_with_name(PICO_DEFAULT_LED_PIN, "On-board LED"));
```

## Initialize the LED GPIO Pin.

Initialize the on-board LED GPIO and set it as an output pin.  This could have been included in the `main()` function but it is better to separate out initialization into individual functions.  This keeps your code organized and easy to extend as your project grows.

``` c
void led_init(void)
{
    gpio_init(PICO_DEFAULT_LED_PIN);
    gpio_set_dir(PICO_DEFAULT_LED_PIN, GPIO_OUT);
}
```

## Set the LED State.

As with `led_init()` I could have placed this code inline in the main loop.  As I intend to grow this example as these blog posts progress it is better to start with a well organized layout.  If `led_on` is `true` set the LED pin to high which turns on the LED, otherwise set the pin to low which turns it off.

``` c
void led_set(bool led_on)
{
    gpio_put(PICO_DEFAULT_LED_PIN, led_on);
}
```

## Main.

Now we get to the meat of this program.  It initializes the standard I/O functions, initializes the GPIO pin used to toggle the LED, and then loops forever performing the following functions:

1. Display a message to `stdout` which can be viewed by connecting to the USB port with a serial terminal.
2. Set the default GPIO pin to high, turning on the on-board LED.
3. Sleep for one second.
4. Set the default GPIO pin to low, turning off the on-board LED.
5. Sleep for another second.

While we have a `return` statement in the `main()` function astute readers will realize we will never reach it.  I could have left it off but I didn't.

``` c
int main()
{
    stdio_init_all();
    led_init();

    while (true) {
        puts("Hello from the Pico 2 basic CMake project!");

        led_set(true);
        sleep_ms(LED_DELAY_MS);

        led_set(false);
        sleep_ms(LED_DELAY_MS);
    }

    return 0;
}
```

# Building the Project.

If you're familiar with `CMake`, you know how to build a project, but if you aren't here are the steps required.

## Configure the Project.

From the project source directory, `~/src/pico2_basic_cmake_project`, in my case, use the following commands to create a `build` directory and configure the build project:

``` shell
mkdir build
cd build
cmake ..
```

This will cause `CMake` to read the project from `CMakeLists.txt` in the parent directory and create the `Makefile` and other necessary files to build the project.  The last few lines when I run the command are:

``` text
-- Configuring done (0.5s)
-- Generating done (0.0s)
-- Build files have been written to: ~/src/pico2-basic-cmake-project/build
```

## Compile the Project.

To build the project we just run `make` from the build directory.  You can also use `cmake --build .` if you like.  For this small project, it doesn't make any difference.  The last couple of lines from compiling the project are:

``` text
[100%] Linking CXX executable pico2_basic_cmake.elf
[100%] Built target pico2_basic_cmake
```

# Load the Project On a Pico 2.

To load the project on a Pico 2 follow these steps:

1. Hold the `BOOTSEL` button on the Pico 2 and insert a USB cable.  This will put the microcontroller into USB mass storage mode and allows you to load software to the storage on the board.
2. There are two options for installing the compiled code on the microcontroller.  You can copy the `pico2_basic_cmake.uf2` file to the drive that was mounted by your OS, or use `picotool`, which is installed with the SDK.

I prefer using `picotool` as it provides more control over installing the software.  The command to load the binary on the Pico 2 I use is:

``` shell
picotool load -fxv pico2_basic_cmake.uf2
```

The flags with the command I use are:

- `f`: Force a device not in BOOTSEL mode to reset so the binary can be loaded.
- `x`: Attempt to execute the downloaded file as a program after loading.
- `v`: Verify the data was written correctly.

And the output from that command is:

``` text
The device was asked to reboot into BOOTSEL mode so the command can be executed.

Family ID 'rp2350-arm-s' can be downloaded in absolute space:
  00000000->02000000
Loading into Flash:   [==============================]  100%
Verifying Flash:      [==============================]  100%
  OK

The device was rebooted to start the application.
```

After the `picotool` command finished the on-board LED starts flashing in the expected one second on, one second off pattern.  And using `screen` I can see the expected message being printed on the serial terminal:

``` text
$ screen /dev/ttyACM0 115200
Hello from the Pico 2 basic CMake project!
Hello from the Pico 2 basic CMake project!
Hello from the Pico 2 basic CMake project!
```


# Final Thoughts.

Not much exciting or new to learn in this post, but I wanted to cover the basics about creating a project to compile and run on the Raspberry Pi Pico 2 microcontroller.  The next few posts will be more interesting as I'll be extending the simple project to support the Pico 1 and Pico 2 boards from the same project files.

# Sample Files.

The sample files mentioned in this post can be viewed on [GitHub](https://github.com/n0bml/blog-samples/tree/main/pico2-basic-cmake-project).

# Pico 2 ARM and RISC-V Development Series Links.

1. [Raspberry Pi Pico 2 ARM Development Environment Setup](https://n0bml.github.io/pico2-arm-dev-setup/).
1. [Raspberry Pi Pico 2 RISC-V Development Environment Setup](https://n0bml.github.io/pico2-riscv-dev-setup/).
1. Raspberry Pi Pico 2 Basic CMake Project (this post).
1. Raspberry Pi Pico 2 Extend Project for Pico 1 and Pico 2 (TBD).
1. Raspberry Pi Pico 2 Extend Project for Pico 2 RISC-V CPU (TBD).
1. Raspberry Pi Pico 2 Universal UF2 Project (TBD).
