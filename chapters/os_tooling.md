# Offline Compilation of OpenCL Kernel Sources

Aside from online compilation during the application execution, OpenCL kernel sources can be compiled offline into binaries that can be loaded into the drivers using special API (e.g. `clCreateProgramWithBinary` or `clCreateProgramWithIL`).

This section described available open source tools for offline compilation of OpenCL kernels.

## Open Source Tools

* [clang](https://clang.llvm.org/) is a compiler front-end for the C/C++ family of languages including OpenCL C and C++ for OpenCL that can compile into an executable binary (e.g. AMDGPU), or a portable binary (e.g. SPIR). It is part of [the LLVM compiler infrastructure project](https://llvm.org/), and there is information regarding [OpenCL kernel language support and standard headers](https://clang.llvm.org/docs/UsersManual.html#opencl-features).
* [libclc](https://libclc.llvm.org/) is a generic and portable implementation of OpenCL builtin function libraries for OpenCL 1.1 - and some functions from later versions of OpenCL can be found there too.
* [SPIRV-LLVM Translator](https://github.com/KhronosGroup/SPIRV-LLVM-Translator) is a library and __llvm-spirv__ tool for translating between LLVM IR and SPIR-V. 
* [clspv](https://github.com/google/clspv) compiler and [clvk](https://github.com/kpet/clvk) runtime layer enable OpenCL applications to be executed with Vulkan drivers.
* [SPIR-V Tools](https://github.com/KhronosGroup/SPIRV-Tools) provide a set of utilities to process SPIR-V binaries including __spirv-opt__ optimizer, __spirv-link__ linker, __spirv-dis__/__spirv-as__ (dis-)assembler, and __spirv-val__ validator.

### Compiling Kernels to SPIR-V

Available open source tools can be used to perform full compilation from OpenCL kernel sources into SPIR-V.

<p align="center">
<br>
<img src="../images/opencl_to_spirv_tooling.jpg">
<br> <br>
  <b>Offline compilation flow for OpenCL kernels into SPIR-V</b>
<br> <br>
</p>

#### Examples

If you want to try the above compilation flow for yourself, after installing the tools, you can use the following commands.

##### Compile for OpenCL runtime

__(i)__ Compiling OpenCL C/C++ for OpenCL file into intermediate LLVM IR (for 32 bit targets) formats:

```
clang -cl-std=cl1.2 -c -target spir -O0 -emit-llvm -o test.bc test.cl
```

To compile C++ for OpenCL source pass `-cl-std=clc++` or use the following file extension `.clcpp`.

```
clang -cl-std=clc++ -c -target spir -O0 -emit-llvm -o test.bc test.cl
clang -c -target spir -O0 -emit-llvm -o test.bc test.clcpp
```
If debugging support is needed  can be obtained `-g` flag can be passed in clang invocation:

```
clang -c test.cl -target spir -o test.bc -g
```

__Note:__ In clang releases up to 12.0, calling most builtin functions requires an extra flag to be passed to clang `-Xclang -finclude-default-header`. Refer to the release documentation for more details.

__(ii)__ Converting LLVM IR into SPIR-V. 

```
llvm-spirv test.bc -o test.spv
```

__Note:__ converting optimized IR is currently unsupported. Therefore, a stand-alone __spirv-opt__ tool can be used to optimize SPIR-V modules.

(iii) Linking multiple modules can be done on LLVM IR level by passing multiple `.bc` files to __llvm-link__.

```
llvm-link test1.bc test2.bc -o app.bc
llvm-spirv app.bc -o app.spv
```
Alternatively, __spirv-link__ can be used:

```
spirv-link -o app.spv test1.spv test2.spv
```

(iv) Once the SPIR-V binary is produced, it can be loaded in OpenCL applications using the `clCreateProgramWithIL` API call.

##### Compile for Vulkan runtime

Compiling OpenCL sources directly to SPIR-V binary format:

```
clspv test.cl -o test.spv
```

The SPIR-V binary files can be further optimized or linked using __spirv-opt__ and __spirv-link__ respectively before loading into Vulkan runtime (TODO: API call).

