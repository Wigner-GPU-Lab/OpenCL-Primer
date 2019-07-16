# Getting started with OpenCL on Microsoft Windows

OpenCL is not native to the Windows operating system, and as such isn't supported across the board of UWP (Universal Windows Platform) platforms (XBox, Hololens, IoT, PC) and the Microsoft Store. Windows as a traditional content creating platform however can build and execute OpenCL applications when built as traditional Win32 GUI or console applications.

_(NOTE: Nothing prevents from using advanced WinRT capabilities to use the ICD in packaged applications (APPX/MSIX), however such applications will most certainly not be able to load a functioning OpenCL runtime on platforms other than PC and a few IoT devices.)_

In this guide we're going to use the following tools:

- C/C++ compiler (MSVC & LLVM)
- CMake (Cross-platform Make)
- Git
- Vcpkg (Cross-platform pacakge management)
- The command-line
- Visual Studio Code

Steps will be provided to obtain a minimal and productive environment.

## Installation

Because most tools we'll be using are not found inside the Microsoft Store, we'll be installing dependencies from various places.

_(NOTE: The author **is** familiar with the Chocolatey project, however have found that the end-user UX on this particular scenario is not worth the effort. Far better UX could be achieved if these few tools were packaged as MSIX and distributed via the Microsoft Store.)_

### C/C++ compiler

From https://visualstudio.microsoft.com/downloads/ select [Build Tools for Visual Studio 2019](https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16)

![image](imgs/BuildToolsForVS2019.png)

and launch the installer. When prompted which components to install, select the feature pack named "Desktop development with C++".

![image](imgs/DesktopDevWithCpp.png)

Among the individual features list, select `C++ CMake tools for Windows` and `Clang compiler for Windows` for convenience. The prior will install CMake, our choice of cross-platform build system, while the latter will be useful when we wish to test natively, if our host code conforms closer with ISO standards.

### Git

You most likely already have Git installed. If not, we'll install it, because this is what Vcpkg uses to keep it's repository up-to-date.

Visiting https://git-scm.com/download/win should immediately kick off the download of the native binary installer for your system.

_**TODO:** step-by-step guide through the installer._

### Vcpkg

UX for obtaining dependencies for C/C++ projects has improved dramatically in the past few years. This guide will make use of [Vcpkg](https://github.com/microsoft/vcpkg.git), a community maintained repo of build scripts for a rapidly growing number of open-source libraries.

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
.\vcpkg.exe install opencl
```

### Visual Studio Code

While the compiler installs, you may also fetch the installer of Visual Studio Code from code.visualstudio.com

![imgage](imgs/CodeWebInstall.png)

## Building on the command-line

To build applications on the command-line on Windows, one needs to open a `Developer Command Prompt for VS 2019`. This can be done in the Start Menu.

- _(NOTE: unlike compilers native to *nix OS flavors, MSVC relies on a few environmental variables to function properly. A Developer Command Prompt is nothing more than an ordinary shell with `vcvarsall.bat` invoked which sets the env vars.)_
- _(NOTE 2: even though `cmake.exe` and our build system driver `ninja.exe` isn't on the PATH by default, they are inside a Developer Command Prompt.)_

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

#ifdef _WIN32
    if (CL_err == CL_SUCCESS)
        printf_s("%u platform(s) found:\n", numPlatforms);
    else
        printf_s("clGetPlatformIDs(%i)\n", CL_err);
#else
    if (CL_err == CL_SUCCESS)
        printf("%u platform(s) found:\n", numPlatforms);
    else
        printf("clGetPlatformIDs(%i)\n", CL_err);
#endif

    return 0;
}
```

The CMake build script for this application that builds it as an ISO C11 app with most sensible compiler warnings turned on looks like:

```cmake
cmake_minimum_required(VERSION 3.7) # 3.1 << C_STANDARD 11
                                    # 3.7 << OpenCL::OpenCL
project(HelloOpenCL LANGUAGES C)

find_package(OpenCL REQUIRED)

add_executable(${PROJECT_NAME} Main.c)

target_link_libraries(${PROJECT_NAME} PRIVATE OpenCL::OpenCL)

set_target_properties(${PROJECT_NAME} PROPERTIES C_STANDARD 11
                                                 C_STANDARD_REQUIRED ON
                                                 C_EXTENSIONS OFF)

target_compile_options(${PROJECT_NAME} PRIVATE $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive->)
```

## Building within Visual Studio

## Building within Visual Studio Code