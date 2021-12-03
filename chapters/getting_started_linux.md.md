# Getting started with OpenCL on Ubuntu Linux

OpenCL is included in all the main distributions of Linux. In this guide we choose Ubuntu as one of the most widespread variants, but all of the distributives have some sort of centralized system of package maintaining and OpenCL realizations can be installed from there.

In this guide we're going to use the following tools:

- C/C++ compiler (gcc & Clang)
- CMake (Cross-platform Make)
- Git
- Vcpkg (Cross-platform pacakge management)
- The command-line
- Visual Studio Code

Steps will be provided to obtain a minimal and productive environment.

## Installation

In this guide we'll be using latest (at the time of writing) Ubuntu 20.04 LTS. Installation for the most part will happen via [apt (Advanced Packaging Tool)](https://en.wikipedia.org/wiki/APT_(software)), the definitive command-line tool for installing software on Debian-base Linux distributives. It is installed with the system automatically.

_(NOTE: installation commands should be issued with root access, so that use of privilege-escalation by `sudo` is mandatory.)_

First update your system. Open your favorite shell in your favorite terminal and run:

``` bash
sudo apt update
sudo apt upgrade
```

### C/C++ compiler

Open your favorite shell in your favorite terminal and run:

``` bash
sudo apt install build-essential -y
```

This will install canonical GNU compiler set `g++/GNU 9.3`, debugger `gdb`, as well as some tools (e.g. `make`, `binutils`) and libraries for compilation and linking of programs.

Another popular choice is the usage of `Clang` compiler, based on intermediate representation of the C-like languages on `LLVM` (originally Low Level Virtual Machine). To use it open your favorite shell in your favorite terminal and run:

``` bash
sudo apt install clang -y
```

This will install `Clang 10` and `LLVM 10`.

### Git

You most likely already have Git installed. If not, we'll install it, because this is what is needed to keep your repositories up-to-date. Open your favorite shell in your favorite terminal and run:

``` bash
sudo apt install git -y
```

### CMake

If you do not have CMake installed yet, we'll install it, as it's the primary supported build system, and it will make our builds much simpler. Open your favorite shell in your favorite terminal and run:

``` bash
sudo apt install cmake ninja-build -y
```

### Visual Studio Code

While IDEs are highly opinionated, for an IDE with widespared adoption and small footprint, VS Code has everything (and more) to get off the ground. It should be installed differently, though. The first way is to use GUI of `Ubuntu Software`, where you can search for `code` and install it. Whether you are a fan of command-line tools, use the second method: open your favorite shell in your favorite terminal and run

``` bash
sudo snap install --classic code
```

### OpenCL-SDK

To build native OpenCL applications, one will minimally need:
- C or C++ compiler
- The OpenCL headers
  - The C and optionally the C++ headers
- An Installable Client Driver (ICD) Loader
  - Shared objects library (libOpenCL.so)
  - Export library (libOpenCL.a)

The SDK can be installed from a package manager like Vcpkg or can be built on-demand from the canonical repositories. Git submodules of the SDK include the headers, both C and C++, and Khronos canonical ICD loader, that are built and installed as parts of SDK.

#### GitHub

If you want to build an SDK from source, the recommended way is cloning the OpenCL-SDK repo. First you should install the dependencies of the SDK.

``` bash
sudo apt install libstb-dev libsfml-dev libglew-dev libtclap-dev ruby doxygen -y
git clone https://github.com/KhronosGroup/OpenCL-SDK.git --recursive
cmake -D CMAKE_INSTALL_PREFIX=./OpenCL-SDK/install -B ./OpenCL-SDK/build -S ./OpenCL-SDK
cmake --build OpenCL-SDK/build --config Release --target install
```

#### Vcpkg

UX for obtaining dependencies for C/C++ projects has improved dramatically in the past few years. This guide will make use of [Vcpkg](https://vcpkg.io/en/index.html), a community maintained repo of build scripts for a rapidly growing number of open-source libraries.

Navigate to the folder where you want to install Vcpkg and issue on the command-line (almost any shell):

``` bash
sudo apt install curl zip unzip tar -y
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
./bootstrap-vcpkg.sh
```

This should build the Vcpkg command-line utility that will take care of the all installations. The utility let's us discover the OpenCL SDK in its repository (beside other packages mentioning OpenCL).

``` bash
./vcpkg search opencl
...
opencl               2.2 (2017.07.... C/C++ headers and ICD loader (Installable Client Driver) for OpenCL
```

We can install it by running:

``` bash
./vcpkg install opencl
```

To build SDK you also need some dependencies. Install them by

``` bash
sudo apt install libudev-dev libx11-dev libxrandr-dev libgl-dev libxmu-dev libglu1-mesa-dev ruby doxygen -y
./vcpkg install sfml tclap glm glew
```

_(Note: if you are targeting 64-bit ARM, use `--triplet=arm64-linux` instead. For more information on triplets, refer to [Triplet Files](https://vcpkg.io/en/docs/users/triplets.html) in the Vcpkg docs.)_

## Compiling on the command-line

### Invoking the compiler manually

Compilers native to *nix OS flavors work just out of the box. Then navigate to the folder where you wish to build your application. Our application will have a single `Main.c` source file:

```c
// C standard includes
#include <stdio.h>

// OpenCL includes
#include <CL/cl.h>

int main()
{
    cl_int CL_err = CL_SUCCESS;
    cl_uint numPlatforms = 0;

    CL_err = clGetPlatformIDs( 0, NULL, &numPlatforms );

    if (CL_err == CL_SUCCESS)
        printf("%u platform(s) found\n", numPlatforms);
    else
        printf("clGetPlatformIDs(%i)\n", CL_err);

    return 0;
}
```

Then invoke the compiler to build our source file as such:

- SDK installed from Vcpkg (uses static linking with OpenCL library `libOpenCL.a`)

      gcc -Wall -Wextra -D CL_TARGET_OPENCL_VERSION=100 -I <SDKROOT>/external/OpenCL-Headers/ Main.c -o HelloOpenCL -lOpenCL -L <VCPKGINSTALLROOT>/packages/opencl_x64-linux/lib -pthread -ldl

- SDK built from source (uses dynamic linking with OpenCL library `libOpenCL.so`)

      gcc -Wall -Wextra -D CL_TARGET_OPENCL_VERSION=100 -I <SDKROOT>/external/OpenCL-Headers/ Main.c -o HelloOpenCL -lOpenCL -L <SDKINSTALLROOT>/lib -Wl,-rpath=<SDKINSTALLROOT>/lib

What do the command-line arguments mean?

- `-Wall -Wextra` turns on all warnings (highest sensible level)
- `-D` instructs the preprocessor to create a define with NAME:VALUE
  - `CL_TARGET_OPENCL_VERSION` enables/disables API functions corresponding to the defined version. Setting it to 100 will disable all API functions in the header that are newer than OpenCL 1.0
- `-I` sets additional paths to the include directory search paths
- `Main.c` is the name of the input source file
- `-o` sets the name of the output executable (default would be `a.out`)
- `-l` instructs linker to link library `OpenCL`
- `-L` sets additional paths to the library directory search paths during linking
- `-pthread` links with `phthread` library
- `-ldl` links with dynamic loader library
- `-Wl` sends the following options to the linker and
- `-rpath` sets hardcoded additional paths to the library directory search paths during execution of the binary

Running our executable by `./HelloOpenCL` either prints the number of platforms found or an error code which is often the result of corrupted or absent runtime installations.

### Automating the build using CMake

The CMake build script for this application that builds it as an ISO C11 app with most sensible compiler warnings turned on looks like:

``` cmake
cmake_minimum_required(VERSION 3.1) # 3.1 << C_STANDARD 11

project(HelloOpenCL LANGUAGES C)

find_package(OpenCL CONFIG REQUIRED)

add_executable(${PROJECT_NAME} Main.c)

target_link_libraries(${PROJECT_NAME} PRIVATE OpenCL::OpenCL)

set_target_properties(${PROJECT_NAME} PROPERTIES C_STANDARD 11
                                                 C_STANDARD_REQUIRED ON
                                                 C_EXTENSIONS OFF)

target_compile_definitions(${PROJECT_NAME} PRIVATE CL_TARGET_OPENCL_VERSION=100)

target_compile_options(${PROJECT_NAME}
  PRIVATE
    $<$<OR:$<CXX_COMPILER_ID:GNU>,$<CXX_COMPILER_ID:Clang>>:
        -Wall     # Turn on all warnings
        -Wextra   # Turn on even more warnings
        -pedantic # Turn on strict language conformance
    >
    $<$<CXX_COMPILER_ID:MSVC>:
        /W4          # Turn on all (sensible) warnings
        /permissive- # Turn on strict language conformance
    >
)
```

What does the script do?
- Give a name to the project and tell CMake to only look for a C compiler (default is to search for a C and a C++ compiler)
- Look for an OpenCL SDK and fail if not found
  - The `CONFIG` part for the time being is important. For more details, refer to the [CMake Build-System Support](./cmake_build-system_support.md) chapter.
- Specify our source files and name the executable
- Specify dependency to the SDK (not just linkage)
- Set language properties to all source files of our application
- Set the OpenCL version target to control API coverage in header
- Because CMake cannot handle warning levels portably, set compiler specific flags. Guard it with a generator expression (terse conditionals), so other compilers will not pick up such non-portable flags.

To invoke this script, place it next to our `Main.c` file in a file called `CMakeLists.txt`. Once that's done, cmake may be invoked the following way to generate Ninja makefiles in the advised out-of-source fashion into a subfolder named `build`:

- SDK installed from Vcpkg

      cmake -S . -B ./build -D CMAKE_TOOLCHAIN_FILE=<VCPKGROOT>/scripts/buildsystems/vcpkg.cmake

- SDK built from source

      cmake -S . -B ./build -D CMAKE_PREFIX_PATH=<SDKINSTALLROOT>

Which will output something like

```
-- The C compiler identification is GNU 9.3.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/ivan/OCL/build
```

To kick off the build, one may use CMakes build driver:

``` bash
cmake --build ./build --config Release
```

Once build is complete, we can run the program by typing:

``` bash
./build/HelloOpenCL
```

## Building within Visual Studio Code

To have a decent developer experience in the IDE, we will need to install a few extensions. (On how to install extensions, refer to the [corresponding docs](https://code.visualstudio.com/docs/editor/extension-gallery).)

- C/C++ by Microsoft
- CMake
- CMake Tools
- CMake Test Explorer
- OpenCL

These extensions will help us author, navigate, build, test, debug code. The OpenCL extension provides syntax highlight for OpenCL device code and provide in-editor documentation for API functions.