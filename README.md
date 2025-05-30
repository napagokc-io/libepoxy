![Ubuntu](https://github.com/napagokc-io/libepoxy/workflows/Ubuntu/badge.svg)
![macOS](https://github.com/napagokc-io/libepoxy/workflows/macOS/badge.svg)
![MSVC Build](https://github.com/napagokc-io/libepoxy/workflows/MSVC%20Build/badge.svg)
![MSYS2 Build](https://github.com/napagokc-io/libepoxy/workflows/MSYS2%20Build/badge.svg)
[![License: MIT](https://img.shields.io/badge/license-MIT-brightgreen.svg)](https://opensource.org/licenses/MIT)

# Epoxy

Epoxy is a library for handling OpenGL function pointer management for you.

It hides the complexity of `dlopen()`, `dlsym()`, `glXGetProcAddress()`,
`eglGetProcAddress()`, etc. from the app developer, with very little
knowledge needed on their part. They get to read GL specs and write
code using undecorated function names like `glCompileShader()`.

Don't forget to check for your extensions or versions being present
before you use them, just like before! We'll tell you what you forgot
to check for instead of just segfaulting, though.

## Features

* Automatically initializes as new GL functions are used
* GL 4.6 core and compatibility context support
* GLES 1/2/3 context support
* Knows about function aliases so (e.g.) `glBufferData()` can be used with both `GL_ARB_vertex_buffer_object` implementations and GL 1.5+ implementations
* EGL, GLX, and WGL support
* Can be mixed with non-epoxy GL usage

## Building with Meson (Recommended)

```sh
mkdir _build && cd _build
meson setup ..
ninja
sudo ninja install
```

## Building with CMake (Alternative)

```sh
# Create and navigate to build directory
mkdir -p build_cmake && cd build_cmake

# Configure with CMake
cmake ..

# Build
cmake --build .

# Install (optional)
sudo cmake --install .
```

CMake Options:
* `-DBUILD_SHARED_LIBS=ON` - Build shared library instead of static
* `-DENABLE_GLX=OFF` - Disable GLX support
* `-DENABLE_EGL=OFF` - Disable EGL support
* `-DENABLE_X11=OFF` - Disable X11 support
* `-DBUILD_APPLE_FRAMEWORK=ON` - Build as a native Apple Framework (macOS/iOS)
* `-DBUILD_TESTS=ON` - Enable tests (when they're implemented)

## Dependencies

### For Debian/Ubuntu
* meson
* libegl1-mesa-dev

### For macOS

Using Homebrew:
```sh
# For Meson build
brew install ninja pkg-config
python3 -m pip install meson

# For CMake build
brew install cmake ninja
```

Using MacPorts:
```sh
# For Meson build
sudo port install pkgconfig ninja meson

# For CMake build  
sudo port install cmake ninja
```

## Building on macOS

### Using Meson

```sh
# Create build directory
mkdir -p build && cd build

# Configure with Meson
meson setup ..

# Build the library
ninja

# Install (optional)
sudo ninja install

# Alternatively, use the provided CI script (from the repo root directory)
CC=clang ./.github/scripts/epoxy-ci-osx.sh

# You can also pass build options to the script
CC=clang ./.github/scripts/epoxy-ci-osx.sh -Dglx=no
```

### Using CMake

```sh
# Create build directory
mkdir -p build_cmake && cd build_cmake

# Configure with CMake
cmake ..

# Build
cmake --build .

# Install (optional)
sudo cmake --install .
```

On macOS, libepoxy uses the native OpenGL framework. The build doesn't require X11 or EGL, 
though these can be enabled with build options.

The test suite has additional dependencies depending on the platform
(X11, EGL, a running X Server).

## Using libepoxy in Your Projects

### With CMake

```cmake
# In your project's CMakeLists.txt
cmake_minimum_required(VERSION 3.18)
project(my_opengl_app)

find_package(epoxy REQUIRED)

add_executable(my_app main.c)
target_link_libraries(my_app PRIVATE epoxy::epoxy)
```

### With pkg-config

```sh
cc `pkg-config --cflags --libs epoxy` -o myapp main.c
```

## Switching your code to using epoxy

It should be as easy as replacing:

```cpp
#include <GL/gl.h>
#include <GL/glx.h>
#include <GL/glext.h>
```

with:

```cpp
#include <epoxy/gl.h>
#include <epoxy/glx.h>
```

As long as epoxy's headers appear first, you should be ready to go.
Additionally, some new helpers become available, so you don't have to
write them:

`int epoxy_gl_version()` returns the GL version:

* 12 for GL 1.2
* 20 for GL 2.0
* 44 for GL 4.4

`bool epoxy_has_gl_extension()` returns whether a GL extension is
available (`GL_ARB_texture_buffer_object`, for example).

Note that this is not terribly fast, so keep it out of your hot paths,
ok?

## Why not use libGLEW?

GLEW has several issues:

* Doesn't know about aliases of functions (There are 5 providers of
  `glPointParameterfv()`, for example, and you don't want to have to
  choose which one to call when they're all the same).
* Doesn't support OpenGL ES.
* Has a hard-to-maintain parser of extension specification text
  instead of using the old .spec file or the new .xml.
* Has significant startup time overhead when `glewInit()`
  autodetects the world.
* User-visible multithreading support choice for win32.

The motivation for this project came out of previous use of libGLEW in
[piglit](http://piglit.freedesktop.org/). Other GL dispatch code
generation projects had similar failures. Ideally, piglit wants to be
able to build a single binary for a test that can run on whatever
context or window system it chooses, not based on link time choices.

We had to solve some of GLEW's problems for piglit and solving them
meant replacing every single piece of GLEW, so we built
piglit-dispatch from scratch. And since we wanted to reuse it in
other GL-related projects, this is the result.

## Known issues when running on Windows

The automatic per-context symbol resolution for win32 requires that
epoxy knows when `wglMakeCurrent()` is called, because `wglGetProcAddress()`
returns values depend on the context's device and pixel format. If
`wglMakeCurrent()` is called from outside of epoxy (in a way that might
change the device or pixel format), then epoxy needs to be notified of
the change using the `epoxy_handle_external_wglMakeCurrent()` function.

The win32 `wglMakeCurrent()` variants are slower than they should be,
because they should be caching the resolved dispatch tables instead of
resetting an entire thread-local dispatch table every time.
