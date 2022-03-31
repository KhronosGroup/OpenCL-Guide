# CMake Build System Support

The Khronos OpenCL ecosystem currently dominantly uses CMake as tool of choice for build automation and unit testing. OpenCL has had facilities for building OpenCL applications long before the the Khronos-hosted OpenCL-SDK manifested.

## Dependency detection

CMake has two package detection mechanisms: Find Modules and Package Config files. The prior is for packages typically built using CMake-unaware tooling while the latter provide 1st class integration into CMake build definitions. CMake has had Find Module support for OpenCL for a long time now while the OpenCL SDK added Package Config support, which is encourage for new OpenCL projects. For more information on [Search Modes](https://cmake.org/cmake/help/latest/command/find_package.html#search-modes), refer to the CMake docs.

### Find Module

For the latest documentation on this method, refer [CMake's docs](https://cmake.org/cmake/help/latest/module/FindOpenCL.html).

#### CMake 3.1+

`FindOpenCL.cmake` has been a part of CMake since version 3.1.

```cmake
cmake_minimum_required(VERSION 3.1)
project(MyApp LANGUAGES C)
find_package(OpenCL REQUIRED)
add_executable(${PROJECT_NAME} Main.c)
target_include_directories(${PROJECT_NAME} PRIVATE ${OpenCL_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} PRIVATE ${OpenCL_LIBRARIES})
```

Builds a simple OpenCL application. The `find_package(OpenCL REQUIRED)` function call will run the script CMake-hosted `FindOpenCL.cmake` script which will look for typical vendor SDK installations. The user may guide detection using both CMake variables and environmental variables.

#### CMake 3.7+

CMake 3.7 added the more modern `IMPORTED` target `OpenCL::OpenCL` which bundles all the necessary compiler switches in one entity.

```cmake
cmake_minimum_required(VERSION 3.7)
project(MyApp LANGUAGES C)
find_package(OpenCL REQUIRED)
add_executable(${PROJECT_NAME} Main.c)
target_link_libraries(${PROJECT_NAME} PRIVATE OpenCL::OpenCL)
```

> Intention is that Kitware's CMake-hosted `FindOpenCL.cmake` with time export the same imported targets as the Khronos-hosted Package Config scripts.

### Package Config

The Khronos-hosted [OpenCL SDK](https://github.com/KhronosGroup/OpenCL-SDK) improves on the Find Module approach by making detection more robust and adding more control over detecting parts of the required components.

```cmake
cmake_minimum_required(VERSION 3.0)
project(MyApp LANGUAGES C)
find_package(OpenCL REQUIRED)
add_executable(${PROJECT_NAME} Main.c)
target_link_libraries(${PROJECT_NAME} PRIVATE OpenCL::OpenCL)
```

The script above builds a simple OpenCL application. Note that script is identical to the `IMPORTED` target version, but doesn't require CMake 3.7.

This script using the [Basic Signature](https://cmake.org/cmake/help/latest/command/find_package.html#basic-signature) of `find_package()` is able to detect the minimal set of OpenCL development files using both the Find Module and Package Config search modes. There is one important caveat to using this simplest form: CMake favors Module mode over Config mode and should the user have some GPGPU SDK installed or has OpenCL development files in system default locations, it will always "hide" a custom built OpenCL SDK. Preferring custom packages over system ones is possible with setting the [`CMAKE_FIND_PACKAGE_PREFER_CONFIG`](https://cmake.org/cmake/help/latest/variable/CMAKE_FIND_PACKAGE_PREFER_CONFIG.html) variable on the command-line. _(Do not bake this into scripts, it is user preference!)_

The OpenCL SDK defines the following `IMPORTED` targets as components:

- `OpenCL::Headers` for the C API headers
- `OpenCL::HeadersCpp` for the C++ API headers (depends on the C Headers)
- `OpenCL::OpenCL` for the ICD Loader (depends on the C Headers)
- `OpenCL::Utils` for the C Utility library
- `OpenCL::UtilsCpp` for the C++ Utility library

## Package manager support

When one doesn't consume the SDK from a vendor SDK nor does one build it from source, it's possible to install it via popular package managers.

### Vcpkg

For a complete guide on Vcpkg, refer to the [product page](https://vcpkg.io/en/index.html).

The short summary is that Vcpkg is a Microsoft and community maintained cross-platform open source tool and repository of packages. The packages are called Ports, and the non-metadata part of a Port is a portfile. The portfile is a CMake script (run by the Vcpkg executable using `cmake -P` in [Script Mode](https://cmake.org/cmake/help/latest/manual/cmake.1.html#run-a-script)) which downloads, patches a project if necessary and builds it from source. To author packages one need not know any other scripting language beside CMake. To check how Vcpkg builds and installs the OpenCL SDK, one may consult [OpenCL's portfile](https://github.com/microsoft/vcpkg/blob/master/ports/opencl/portfile.cmake).

To build and install the OpenCL SDK, a typical vcpkg CLI invocation would be:

```
vcpkg install opencl
```

_(For a complete guide on what triplets are, how to select compilers, bitness, static/dynamic library flavors, etc. refer to the [product docs](https://vcpkg.io/en/docs/README.html). Windows users for eg. typically want to add `--triplet=x64-windows` to the command-line or set the `VCPKG_DEFAULT_TRIPLET` environmental variable)_

The CMake script samples above need no changes, build scripts are agnostic to consuming the SDK from the system, from a custom build or from Vcpkg. The only thing needing an update is the CMake command-line invocation. By adding `-D CMAKE_TOOLCHAIN_FILE=<VCPKGROOT>\scripts\buildsystems\vcpkg.cmake -D ` to the CMake command-line we "instruct CMake to use Vcpkg". Explicitly specifying the installed package pool of a triplet is done by adding for eg. `-D VCPKG_TARGET_TRIPLET=x64-windows`.

#### Vcpkg CMake integration

A cornerstone feature of Vcpkg is how it integrates into standard CMake workflow. CMake by default has no knowledge of where to look for our OpenCL SDK package obtained by Vcpkg. On platforms like Windows, there aren't even system include/lib directories to search. We could instruct every package individually where to locate itself, in the case of OpenCL for eg. via providing `-D OpenCL_INCLUDE_DIR=C:\Users\<username>\Source\Repos\vcpkg\installed\x64-windows\include -D OpenCL_LIBRARY=C:\Users\<username>\Source\Repos\vcpkg\installed\x64-windows\debug\lib\OpenCL.lib` on the CMake invocation. This very soon would become tedious.

CMake has a notion of Toolchain Files which can be used to set canonical variables to compiler/linker/etc. executables to override CMake's search mechanism. Toolchain files are loaded very early during configuration (even before the first line of `CMakeLists.txt`). Vcpkg (ab)uses this mechanism not to define a toolchain, but to hijack much of CMake's internal machinery, including finding packages. Vcpkg appends it's own install directories to `CMAKE_PREFIX_PATH` and similar variables so `find_package` calls find packages installed by Vcpkg. Moreover it wraps/overrides the entire `find_package` command, because some libraries need extra rituals to consume. One such library is Boost, which is wrapped [like this](https://github.com/microsoft/vcpkg/blob/193880f0a8d9738a62142dd49aab19069b1ff8d6/scripts/buildsystems/vcpkg.cmake#L743-L756) taking Vcpkg-specific variables and turning them into `Boost_COMPILER` and similar variables before issuing the real `find_package` command.

Vcpkg doesn't monopolize Toolchain Files, by providing `VCPKG_CHAINLOAD_TOOLCHAIN_FILE` one can specify a _real_ toolchain file which will be loaded after Vcpkg's own. And that's all Vcpkg is.