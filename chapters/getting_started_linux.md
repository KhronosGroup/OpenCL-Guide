# Getting started with OpenCL on Ubuntu Linux

OpenCL is included in all the main distributions of Linux. In this guide we choose Ubuntu as one of the most widespread variants, but all of the distributions have some sort of centralized system of package maintaining and OpenCL realizations can be installed from there.

In this guide we're minimally going to use the following tools:

- The command-line
- C/C++ compiler

And depending on which ways one extends the bare minimum:

- CMake (Cross-platform Make)
- Git
- Vcpkg (Cross-platform pacakge management)
- Visual Studio Code

Steps will be provided to obtain a minimal and productive environment.

## Installation

In this guide we'll be using latest (at the time of writing) Ubuntu 20.04 LTS. Installation for the most part will happen via [apt (Advanced Packaging Tool)](https://en.wikipedia.org/wiki/APT_(software)), the definitive command-line tool for installing software on Debian-base Linux distribution. It is installed with the system automatically.

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

### Git

You most likely already have Git installed. If not, we'll install it, because this is what is needed to keep your repositories up-to-date. Open your favorite shell in your favorite terminal and run:

``` bash
sudo apt install git -y
```

### CMake

If you do not have CMake installed yet, we'll install it, as it's the primary supported build system, and it will make our builds much simpler. Open your favorite shell in your favorite terminal and run:

``` bash
sudo apt install cmake -y
```

### Visual Studio Code

While IDEs are highly opinionated, for an IDE with widespread adoption and small footprint, VS Code has everything (and more) to get off the ground. It can be installed in two ways. The first way is to use the `Ubuntu Software` GUI, where you can search for `code` and install it in one click. If you are a fan of command-line tools, use the second method: open your favorite shell in your favorite terminal and run

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

The easiest way to obtain these files is using the system package manager, albeit may not be the latest on non-rolling distributions. Cutting-edge versions may be obtained using C/C++ package managers (like Vcpkg) or can be built on-demand from the canonical GitHub-hosted repositories. Git submodules of the SDK include the headers, both C and C++, and Khronos canonical ICD loader, that are built and installed as parts of SDK.

<details open>
<summary>
<b>APT</b>
</summary>

```
sudo apt install opencl-headers ocl-icd-opencl-dev -y
```
</details>

<details>
<summary>
<b>GitHub</b>
</summary>

If you want to build an SDK from source, the recommended way is cloning the OpenCL-SDK repo. First you should install the dependencies of the SDK.

``` bash
sudo apt install libstb-dev libsfml-dev libglew-dev libglm-dev libtclap-dev ruby doxygen -y
git clone https://github.com/KhronosGroup/OpenCL-SDK.git --recursive
cmake -D CMAKE_INSTALL_PREFIX=./OpenCL-SDK/install -B ./OpenCL-SDK/build -S ./OpenCL-SDK
cmake --build OpenCL-SDK/build --config Release --target install
```
</details>

<details>
<summary>
<b>Vcpkg</b>
</summary>

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
./vcpkg install sfml glm glew tclap
```

_(Note: if you are targeting 64-bit ARM, use `--triplet=arm64-linux` instead. For more information on triplets, refer to [Triplet Files](https://vcpkg.io/en/docs/users/triplets.html) in the Vcpkg docs.)_
</details>

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

<details open>
<summary>
<b>APT</b>
</summary>

    gcc -Wall -Wextra -D CL_TARGET_OPENCL_VERSION=100 Main.c -o HelloOpenCL -lOpenCL

What do the command-line arguments mean?

- `-Wall -Wextra` turns on all warnings (highest sensible level)
- `-D` instructs the preprocessor to create a define with NAME:VALUE
  - `CL_TARGET_OPENCL_VERSION` enables/disables API functions corresponding to the defined version. Setting it to 100 will disable all API functions in the header that are newer than OpenCL 1.0
- `-I` sets additional paths to the include directory search paths
- `Main.c` is the name of the input source file
- `-o` sets the name of the output executable (default would be `a.out`)
- `-l` instructs linker to link library `OpenCL`

</details>

<details>
<summary>
<b>GitHub</b>
</summary>

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
- `-Wl` sends the following options to the linker and
- `-rpath` sets hardcoded additional paths to the library directory search paths during execution of the binary

</details>

<details>
<summary>
<b>Vcpkg</b>
</summary>

    gcc -Wall -Wextra -D CL_TARGET_OPENCL_VERSION=100 -I <SDKROOT>/external/OpenCL-Headers/ Main.c -o HelloOpenCL -lOpenCL -L <VCPKGINSTALLROOT>/packages/opencl_x64-linux/lib -pthread -ldl

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

</details>

Running our executable by `./HelloOpenCL` either prints the number of platforms found or an error code which is often the result of corrupted or absent runtime installations.

### Automating the build using CMake

The CMake build script for this application that builds it as an ISO C11 app with most sensible compiler warnings turned on looks like:

``` cmake
cmake_minimum_required(VERSION 3.1) # 3.1 << C_STANDARD 11

project(HelloOpenCL LANGUAGES C)

find_package(OpenCL REQUIRED)

add_executable(${PROJECT_NAME} Main.c)

target_link_libraries(${PROJECT_NAME} PRIVATE OpenCL::OpenCL)

set_target_properties(${PROJECT_NAME} PROPERTIES C_STANDARD 11
                                                 C_STANDARD_REQUIRED ON
                                                 C_EXTENSIONS OFF)

target_compile_definitions(${PROJECT_NAME} PRIVATE CL_TARGET_OPENCL_VERSION=100)
```

What does the script do?
- Give a name to the project and tell CMake to only look for a C compiler (default is to search for a C and a C++ compiler)
- Look for an OpenCL SDK and fail if not found
  - If detection fails, refer to the [CMake Build-System Support](./cmake_build-system_support.md) chapter.
- Specify our source files and name the executable
- Specify dependency to the SDK (not just linkage)
- Set language properties to all source files of our application
- Set the OpenCL version target to control API coverage in header

To invoke this script, place it next to our `Main.c` file in a file called `CMakeLists.txt`. Once that's done, CMake may be invoked the following way to generate makefiles in the advised out-of-source fashion into a subfolder named `build`:

<details open>
<summary>
<b>APT</b>
</summary>

    cmake -S . -B ./build

</details>

<details>
<summary>
<b>GitHub</b>
</summary>

    cmake -S . -B ./build -D CMAKE_PREFIX_PATH=<SDKINSTALLROOT>

</details>

<details>
<summary>
<b>Vcpkg</b>
</summary>

    cmake -S . -B ./build -D CMAKE_TOOLCHAIN_FILE=<VCPKGROOT>/scripts/buildsystems/vcpkg.cmake

</details>

<br>

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