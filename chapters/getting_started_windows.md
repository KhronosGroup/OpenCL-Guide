# Getting started with OpenCL on Microsoft Windows

OpenCL is not native to the Windows operating system, and as such isn't supported across the board of UWP (Universal Windows Platform) platforms (XBox, Hololens, IoT, PC) and the Microsoft Store. Windows as a traditional content creating platform however can build and execute OpenCL applications when built as traditional Win32 GUI or console applications.

_(NOTE: Nothing prevents from using advanced WinRT capabilities to use the ICD in packaged applications (APPX/MSIX), however such applications will most certainly not be able to load a functioning OpenCL runtime on platforms other than PC and a few IoT devices.)_

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

In this guide we'll be using latest (at the time of writing) Windows 10. Installation for the most part will happen via [winget](https://docs.microsoft.com/en-us/windows/package-manager/winget/), the 1st party command-line tool for installing software on Windows. If for whatever reason you do not have the `winget` client on your PATH, it is bundled with the [App Installer](https://apps.microsoft.com/store/detail/9NBLGGH4NNS1?hl=en-us&gl=US) package in the Microsoft Store. Updating this component should also install the command-line tool.

_(NOTE: installation commands if not issued from a shell with Administrator privileges, UAC prompts will pop up. If you are in an automated scenario where that's an issue, make sure to work in such a shell.)_

> After installing everything, open a new shell, otherwise the tools will not be available on the PATH for convenient invocation.

### C/C++ compiler

Open your favorite shell in your favorite terminal and issue:

```
winget install "Visual Studio Build Tools 2022"
```

#### Command-line component selection

In automated scenarios with no GUI available, please consult the [VS docs](https://docs.microsoft.com/en-us/visualstudio/install/command-line-parameter-examples?view=vs-2022) on specifying the required workloads. A minimal configuration which may be imported using the GUI or specfied on the command-line is as:

```powershell
& "C:\Program Files (x86)\Microsoft Visual Studio\Installer\setup.exe" install --passive --norestart --productId Microsoft.VisualStudio.Product.BuildTools --channelId VisualStudio.17.Release --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 --add Microsoft.VisualStudio.Component.VC.Redist.14.Latest
```

> Developers targeting Windows 10 will need to add `--add Microsoft.VisualStudio.Component.Windows10SDK.19041` to the command-line while targeting Windows 11 requires `--add Microsoft.VisualStudio.Component.Windows11SDK.22000`

### Git

You most likely already have Git installed. If not, we'll install it, because this is what Vcpkg uses to keep it's repository up-to-date. Open your favorite shell in your faovrite terminal and issue:

```
winget install Git.Git
```

### CMake

If you do not have CMake installed yet, we'll install it, as it's the primary supported build system, as it will make our builds much simpler. Open your favorite shell in your faovrite terminal and issue:

```
winget install Kitware.CMake
```

### Visual Studio Code

While IDEs are highly opinionated, for an IDE with widespread adoption and small footprint, VS Code has everything (and more) to get off the ground.

```
winget install "Visual Studio Code" --source msstore
```

### OpenCL-SDK

To build native OpenCL applications, one will minimally need:
- C or C++ compiler
- The OpenCL headers
  - The C and optionally the C++ headers
- An Installable Client Driver (ICD) Loader
  - Dynamic library (OpenCL.dll)
  - Export library (OpenCL.lib)

The SDK can be installed from a package manager like Vcpkg or can be built on-demand from their canonical repositories.

<details open>
<summary>
<b>GitHub</b>
</summary>

If you want to build an SDK from source, the recommended way is cloning the OpenCL-SDK repo.

```
git clone --recursive https://github.com/KhronosGroup/OpenCL-SDK.git
cmake -G "Visual Studio 17 2022" -A x64 -T v143 -D CMAKE_INSTALL_PREFIX=./OpenCL-SDK/install -B ./OpenCL-SDK/build -S ./OpenCL-SDK
cmake --build OpenCL-SDK/build --config Release --target install -- /m /v:minimal
```
</details>

<details>
<summary>
<b>Vcpkg</b>
</summary>

UX for obtaining dependencies for C/C++ projects has improved dramatically in the past few years. This guide will make use of [Vcpkg](https://vcpkg.io/en/index.html), a community maintained repo of build scripts for a rapidly growing number of open-source libraries.

Navigate to the folder where you want to install Vcpkg and issue on the command-line (almost any shell):

```
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
.\bootstrap-vcpkg.bat
```

This should build the Vcpkg command-line utility that will take care of the all installations. The utility let's us discover the OpenCL SDK in its repository (beside other packages mentioning OpenCL).

```
.\vcpkg.exe search opencl
...
opencl               2.2 (2017.07.... C/C++ headers and ICD loader (Installable Client Driver) for OpenCL
```

We can install it by issuing:

```
.\vcpkg.exe --triplet=x64-windows install opencl
```

_(Note: if you are targeting 64-bit ARM, use `--triplet=arm64-windows` instead. For more information on triplets, refer to [Triplet Files](https://vcpkg.io/en/docs/users/triplets.html) in the Vcpkg docs.)_

</details>

## Compiling on the command-line

On Windows the default system command interpreter is `cmd.exe`. (This is analogous to `/bin/sh` in Linux.) The default shell however is PowerShell which provides better user-experience overall. (Much like Linux defaults to `/bin/bash`.) PowerShell provides a better CLI experience overall.

### Invoking the compiler manually

Unlike compilers native to *nix OS flavors, MSVC relies on a few environmental variables to function properly. Such environments are referred to as Developer Command Prompt or Developer PowerShell, which are nothing more than ordinary shells with these environmental variables setup.)

#### Developer Command Prompt

To build applications on the command-line using `cmd.exe` on Windows, one needs to open a `Developer Command Prompt for VS 2022`. This can be done in the Start Menu or by invoking `vcvarsall.bat` from the installation folder.

```cmd
"C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvarsall.bat" amd64
```

_(Note: for all combinations of available host-target toolset variables, consult `vcvarsall.bat help`)_

#### Developer PowerShell

Inheriting environments set up by batch scripts is not trivial, hence VS provides a PowerShell module doing the same thing. The module is inside an assembly shipping with VS. To have the utility available in all shells, invoke the following from PowerShell:

```powershell
if (-not (Test-Path ~\Documents\WindowsPowerShell)) { New-Item -Type Directory ~\Documents\WindowsPowerShell }
'Import-Module "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\Common7\Tools\Microsoft.VisualStudio.DevShell.dll"' >> $PROFILE
function devshell { Enter-VsDevShell -InstallPath "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools" -SkipAutomaticLocation -DevCmdArguments "-arch=x64 -no_logo" }

```

Opening a new shell, one now can enter a Developer PowerShell by issuing:

```powershell
devshell -VsInstallPath 'C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\' -DevCmdArguments '-arch=x64 -no_logo'
```

Inside a Developer Command Prompt navigate to the folder where you wish to build your application. Our application will have a single `Main.c` source file:

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
        printf_s("%u platform(s) found\n", numPlatforms);
    else
        printf_s("clGetPlatformIDs(%i)\n", CL_err);

    return 0;
}
```

Then invoke the compiler to build our source file as such:

<details open>
<summary>
<b>GitHub</b>
</summary>

    cl.exe /nologo /TC /W4 /DCL_TARGET_OPENCL_VERSION=100 /I<SDKINSTALLROOT>\include\ Main.c /Fe:HelloOpenCL /link /LIBPATH:<SDKINSTALLROOT>\lib OpenCL.lib

</details>

<details>
<summary>
<b>Vcpkg</b>
</summary>

      cl.exe /nologo /TC /W4 /DCL_TARGET_OPENCL_VERSION=100 /I<VCPKGROOT>\installed\x64-windows\include\ Main.c /Fe:HelloOpenCL /link /LIBPATH:<VCPKGROOT>\installed\x64-windows\lib OpenCL.lib

</details>

What do the command-line arguments mean?

- `/nologo` makes the compiler omit printing a banner to the console
- `/TC` tells it to treat all source files as C
- `/W4` turns on Warning level 4 (highest sensible level)
- `/D` instructs the preprocessor to create a define with NAME:VALUE
  - `CL_TARGET_OPENCL_VERSION` enables/disables API functions corresponding to the defined version. Setting it to 100 will disable all API functions in the header that are newer than OpenCL 1.0
- `/I` sets additional paths to the include directory search paths
- `Main.c` is the name of the input source file
- `/Fe:` sets the name of the output executable (default would be `Main.exe`)
- `/link` allows passing arguments to the linker invoked by the compiler driver (flags of the compiler should not follow this argument)
- `/LIBPATH` sets additional paths to the library directory search paths
- `OpenCL.lib` is the vendor neutral ICD loader to link to

Running our executable `HelloOpenCL.exe` either prints the number of platforms found or an error code which is often the result of corrupted runtime installations.

### Automating the build using CMake

The CMake build script for this application that builds it as an ISO C11 app with most sensible compiler warnings turned on looks like:

```cmake
cmake_minimum_required(VERSION 3.1) # 3.1 << C_STANDARD 11

project(HelloOpenCL LANGUAGES C)

find_package(OpenCL CONFIG REQUIRED)

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
- Because CMake cannot handle warning levels portably, set compiler specific flags. Guard it with a generator expression (terse conditionals), so other compilers will not pick up such non-portable flags.

To invoke this script, place it next to our `Main.c` file in a file called `CMakeLists.txt`. Once that's done, cmake may be invoked the following way to generate Ninja makefiles in the advised out-of-source fashion into a subfolder named `build`:

<details open>
<summary>
<b>GitHub</b>
</summary>

    cmake -A x64 -S . -B .\build -D CMAKE_PREFIX_PATH=<SDKINSTALLROOT>

</details>

<details>
<summary>
<b>Vcpkg</b>
</summary>

      cmake -A x64 -S . -B .\build -D CMAKE_TOOLCHAIN_FILE=<VCPKGROOT>\scripts\buildsystems\vcpkg.cmake

</details>

Which will output something like
```
-- The C compiler identification is MSVC 19.22.27905.0
-- Check for working C compiler: C:/Program Files (x86)/Microsoft Visual Studio620226BuildTools/VC/Tools/MSVC/14.22.27905/bin/Hostx64/x64/cl.exe
-- Check for working C compiler: C:/Program Files (x86)/Microsoft Visual Studio620226BuildTools/VC/Tools/MSVC/14.22.27905/bin/Hostx64/x64/cl.exe -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Looking for CL_VERSION_2_2
-- Looking for CL_VERSION_2_2 - found
-- Found OpenCL: C:/Users/mnagy/Source/Repos/vcpkg/installed/x64-windows/debug/lib/OpenCL.lib (found version "2.2")
-- Looking for pthread.h
-- Looking for pthread.h - not found
-- Found Threads: TRUE
-- Configuring done
-- Generating done
-- Build files have been written to: C:/Users/mnagy/Source/CL-Min1/build
```

To kick off the build, one may use CMakes build driver:

```
cmake --build .\build --config Release
```

Once build is complete, we can run it by typing:

```
.\build\Release\HelloOpenCL.exe
```
