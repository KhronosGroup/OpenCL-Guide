# C++ for OpenCL

The OpenCL working group has transitioned from the original OpenCL C++ kernel language first defined in OpenCL 2.1 to the community developed  [C++ for OpenCL](https://github.com/KhronosGroup/Khronosdotorg/blob/master/api/opencl/assets/CXX_for_OpenCL.pdf) open source front end compiler that provides improved features and compatibility with OpenCL C.

C++ for OpenCL is supported by [Clang](https://clang.llvm.org/docs/UsersManual.html#c-for-opencl) and uses the [LLVM](https://llvm.org/) compiler infrastructure. Its implementation in Clang can be tracked via the [OpenCL Support Page](https://clang.llvm.org/docs/OpenCLSupport.html). It enables developers to use most C++17 features in OpenCL kernels and generates code, through offline compilation, in the SPIR-V intermediate representation that is ingested by an increasing number of OpenCL implementations.

<p align="center">
<br>
<img src="../images/cpp_for_opencl.jpg" width=700 >
<br> <br>
  <b>C++ for OpenCL Brings Together the Capabilities of OpenCL and C++17</b>
<br> <br>
</p>

OpenCL C code is valid and fully compatible with C++ for OpenCL. This enables developers to use a consistent front-end compiler as they incrementally transition from OpenCL C to use C++ features for their applications.

C++ for OpenCL generates SPIR-V 1.0 plus SPIR-V 1.2 where necessary. Experimental support was added in [Clang 9](https://clang.llvm.org/docs/UsersManual.html#cxx-for-opencl) with bug fixes and improvements in [Clang 10](https://releases.llvm.org/10.0.0/tools/clang/docs/ReleaseNotes.html#opencl-kernel-language-changes-in-clang).

You can check out C++ for OpenCL in [Compiler Explorer](https://godbolt.org/z/NGZw9U).
