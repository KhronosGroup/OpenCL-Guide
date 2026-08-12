# OpenCL-OpenGL interop

Both OpenCL and OpenGL have specific extensions targeting resource sharing and synchronizing between the two runtimes. Doing so one may omit fetching data from the device, only to send it immediately back resulting in significant performane gains. Because the way the two APIs work, there are few thing to keep in mind when designing applications that intend interoperating.

## How is it different than using OpenGL compute shaders?

OpenGL compute shaders are slightly more restricted than OpenCL compute kernels. This is also reflected in the duality of the intermediate formats they can be compiled to. When using SPIR-V as an intermediate representation (IR), compute shaders are compiled to the graphics flavor of SIPR-V, which must exhibit structured control flow and must not use pointer arithmetic. These two cannot arise when using GLSL or other traditional shading languages. OpenCL C, being a C-derivate is far more liberal in the expressable language constructs than shading languages and as such requires a more feature complete intermediate representation, the so called compute flavor of SPIR-V. Different compiler infrastructure is required behind the scenes to process these two types of workloads, irrespective of ingesting IR or compiling from source.

Beside the OpenCL ecosystem having far more libraries and utilities tailored toward compute tasks, for applications which are heavier on compute and are graphically less intensive, formulating the majority of the application in a pure compute fashion with a few graphics extensions may be a better solution than having to deal with render pipelines to utilize one pipeline stage almost exclusively.

## Setting up interop

The core of the OpenGL API has remained backward compatible with itself all the way back to it's initial incarnations. This feature of OpenGL imposes some restrictions on how interoperability can be setup.

In layman's terms, OpenCL is the "smarter" API, OpenGL does some part of init unaware of OpenCL, or even before any OpenCL API function has been invoked. Once all the shared resources (buffers and textures) were created in OpenGL, _only then_ is the OpenCL interop context even created. While OpenGL created resources as normal, OpenCL (and only OpenCL) has special functions which take `GLuint` as input to designate which exact OpenGL resource is bing given a corresponding OpenCL handle.

## Using shared resources

The asymmetry in responsibilities is visible in how resources are accessed as well. OpenGL rendering (without further extensions) is conducted as normal, once again only OpenCL has specific functionality to note shared resource usage. Shared resources can only be used, if the device (through a commandqueue) has signaled OpenCL use of the resource via explicit acquire/release semantics using `clEnqueueAcquireGLObjects`/`clEnqueueReleaseGLObjects` functions.

## Synchronizing the two APIs

There are a handful of ways the two APIs may be synchronized depending on how your application is designed and what level of OpenCL-OpenGL interoperabiltiy is supported by the runtimes.

The following sections are practical paraphrases of the OpenCL Extensions specification sections [Synchronizing OpenCL and OpenGL Access to Shared Objects](https://www.khronos.org/registry/OpenCL/specs/3.0-unified/html/OpenCL_Ext.html#cl_khr_gl_sharing__memobjs-synchronizing-opencl-and-opengl-access-to-shared-objects) and changes to this behavior when event sharing is supported described in section [Additions to the OpenCL Extension Specification](https://www.khronos.org/registry/OpenCL/specs/3.0-unified/html/OpenCL_Ext.html#cl_khr_gl_event-additions-to-extension-specification).

### Basic sync

The most basic level of synchronization is when only `cl_khr_gl_sharing` is supported. In such cases the only portable sync pattern uses the most heavyweight sync operations. Rendering in OpenGL and compute in OpenCL shall not overlap and the developer must ensure this using `glFinish()`/`clFinish()`. Both functions signal that all operations have completed in their respective APIs.

_(Note: glFinish() synchronizes the OpenGL client and server too, and as such requries OS intervention. In remote scenarios (such as X-forwarding) this requires network communication as well.)_

### Implicit sync

If `cl_khr_gl_event` is supported, without making use of the added API surface, a faster sync is available is the application is designed in a compatible manner. If the OpenGL context is bound on the thread where acquire/release and compute kernels are enqueued, the OpenCL runtime has a chance to observe the state of the OpenGL context. In such cases, acquiring the OpenGL objects waits for all OpenGL commands to finish that used the acquired resources, _and_ OpenGL calls using these resources which are issued after the release command will not start executing until the effects of release are visible to the OpenGL context.

Implicit sync from the code's perspective resembles that of the previous approach when one does not sync, just flushes the queues instead of finishing them. (Flushing a queue in OpenGL does not involve the OpenGL server.)

_(Note: If in a loop one is calling GL-CL-GL-CL... commands in succession, one blocking sany somewhere will still be required, otherwise such loops on the host may spin faster than rendering and compute commands are processed on the device, leading to spilling the limit of commands in the queues. Blocking can both be done on OpenGL sync objects or OpenCL events.)_

### Explicit sync

When `cl_khr_gl_event` is supported but the context cannot be made current on the thread enqueueing OpenCL commands, one may still sync faster than invoking `glFinish()`/`clFinish()`. Because the OpenCL runtime cannot directly observe the OpenGL context, some channel of information need be made explicit for syncing to occur. As the name suggests, this extension involves events, specifically one is able to create an OpenCL event from an OpenGL sync object.

By mapping a sync object that is enqueue after a render command using some shared resource to an OpenCL event, one can use such events in the call to `clEnqueueAcquireGLObjects` in the event wait list. That way `glFinish()` may be omitted, as OpenCL can explicitly wait on certain parts of the rendering queue to complete. Note than using only this, `clFinish()` strictly speaking is still required.

The corollary to this extension is `GL_ARB_cl_event` which allows syncing in the "reverse direction" by mapping OpenCL event to OpenGL sync objects. That way OpenGL too gets the chance to invoke a lightweight sync operation to make sure that relevant OpenCL operations have completed.