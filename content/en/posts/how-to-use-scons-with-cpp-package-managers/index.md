---
title: "How to use the scons build-system with C++ software package managers (vcpkg, conan)"
date: 2024-12-29T18:00:05+01:00
draft: true
---

## Introduction
As you know from perhaps my previous articles on [how to contribute to Godot](https://paddy-exe.github.io/posts/contributing-to-godot-visual-shader-nodes/),
[creating your own GDExtension](https://paddy-exe.github.io/posts/gdextension-how-to-write-your-first-extension/) or from the official Godot docs,
Godot uses the [SCons build-system](https://scons.org/) to compile the engine code written in C++ to an executable.
SCons tells the compiler specifically HOW to build together that executable.
Another build-system you will have probably heard of is the widely popular and de-facto industry standard CMake.

These build-systems help you not only to compile your own code but also to combine your code together with C++ code others have written as libraries.
These libraries also get updates so if you use several libraries in your projects it can get quite tedious managing everything.
Here come software package manager to your rescue. They let you "install" a library into your project (meaning they download that source code and compile it)
and then hold a record of which libraries with what version you have "installed" in your project.

The best known software package manager for C and C++ are [vcpkg](https://vcpkg.io/en/) and [conan](https://conan.io/).
Both have a lot of features in common and are different in more advanced use-cases in which I won't go into here.
Both have excellent support for CMake since - again - it is the de-facto standard. However, how do you use them with the scons build-system?

## Requirements
If you want to follow along this post, you will need recent versions of following software installed:
- Python
- SCons
- CMake
- C++ compiler

## The manual way
To understand how SCons can generally be used with third-party libraries we will first explore the manual way of integrating libraries into our codebase .
This will later help to understand how it can be done with package managers as well and will show how they help us. So let's create an example project.

> [!INFO]
> No need to recreate the following project structure yourself by hand. You can check it out [here](https://github.com/paddy-exe/manual-library-integration-scons)

Let's consider the following project structure:

```sh
manual-library-integration-scons/ # root folder name
|
+--src/                   # source code of the program
|   |                     # we are building
|   +--helloworld.cpp
|   +--helloworld.h
|
+--sumlib/                # folder of the "external" shared library
|   |
|   +--CMakeLists.txt
|   +--sumlib.cpp
|   +--sumlib.h
|
+--SConstruct             # SCons build file
```

The `sumlib` folder only contains the library code which will be compiled to a dynamic library. Here we have it inside 
our project structure directly, but you can also consider the same procedure when using [git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules).

The contents of the files are purposefully written simple. We are merely creating a class with a static function that
can add two integers together and returns the result. 

```cpp
// sumlib.hpp header file
class SumLib {
public:
    static int add_two(int p_val_a, int p_val_b);
};
```

```cpp
// sumlib.cpp source file
#include "sumlib.hpp"

int SumLib::add_two(int p_val_a, int p_val_b) {
    return p_val_a + p_val_b;
}
```

The more complicated part of this setup is the `CMakeLists.txt` file.

```cmake
# set the minimum required cmake version to compile our library
cmake_minimum_required(VERSION 3.10)

# important for building for both archs on Mac
# will be ignored on non-Apple platforms
set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64" CACHE STRING "" FORCE)

# name the project
project(SumLib)

# set the C++ standard and if it should be required by the compiler to build
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# set a variable for all source files
set(SOURCE_FILES
    sumlib.cpp
)

# set a variable for all header files
set(HEADER_FILES
    sumlib.hpp
)

# set where the cmake should output the compiled library
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib/)

# instruct cmake to compile the header and source files to a shared library
add_library(SumLib SHARED ${HEADER_FILES} ${SOURCE_FILES})
```

Our `src` folder has the following contents.

> [!NOTE]
> Have a look at the line `#include "sumlib/sumlib.hpp"`. The `include` preprocessor has no relative path to the file we are including.
> Generally it is recommended to have only paths from the root of your project and avoid `include` starting with `../` to have it easier when
> you want to restructure your project later on.
> This can be set by your build-system or the IDE you are using (mostly in Visual Studio or XCode)

```cpp
// helloworld.cpp source file
#include "sumlib/sumlib.hpp"
#include <iostream>

int main() {
    std::cout << SumLib::add_two(5, 2);
    std::cout << "\n";
    return 0;
}
```

The SConstruct build file in the root directory for calling SCons looks like this:
```py
#!/usr/bin/env python
import os
import sys

env = Environment()

# Glob search through source files given a specific pattern
sources = Glob("src/*.cpp")

env.Append(CPPPATH=["."])
env.Append(LIBS=["libSumLib"])
env.Append(LIBPATH=["sumlib/lib/"])

env.Append(LINKFLAGS=["-Wl,-rpath," + os.path.abspath("sumlib/lib/")])

# set c++ standard to 11
env.Append(CXXFLAGS=['-std=c++11'])

# Create a runnable program given the source files with all their dependencies
env.Program("helloworld", sources)
```

We are first creating a [construction environment variable](https://scons.org/doc/0.97/HTML/scons-user/c1051.html). 
SCons fills this automatically upon creation with default settings based on our running system. By appending values to this
environment we can configure the build process in which SCons tells the compiler how the program should be built.

Next up, we are collecting all the source files via the [`Glob` function](https://scons.org/doc/latest/HTML/scons-user/apd.html#f-Glob)
using a specific pattern. Since we need every file in the `src` directory we use the wildcard character `*`.

Now, the [`CPPPATH`](https://scons.org/doc/0.92/HTML/scons-user/x537.html) environment variable is for telling the 
compiler where to find the dependencies you are referencing in your files with `include` statements.

The [`LIBS` variable](https://scons.org/doc/1.2.0/HTML/scons-user/a4774.html#cv-LIBS) sets the name of libraries which 
will be linked to your program.

[`LIBPATH`](https://scons.org/doc/1.2.0/HTML/scons-user/a4774.html#cv-LIBPATH) is the variable for the paths where scons 
will search for the libraries you are linking to your program. Since CMake will output the dynamic library into the `lib`
folder we add this path here.

Specifically for macOS we need to set extra arguments for `LINKFLAGS`, as we are adding an extra path to the `rpath` via the
`-rpath` linker flag so macOS knows where the library is when running the program we created. 

> [!INFO]
> `rpath` is the **r**un-time search **path** on Linux and macOS where your executable looks for dynamic libraries. 
> On macOS for instance it is usually located inside the `*.app/Frameworks` folder. 
> You can read more about this topic [here](https://duerrenberger.dev/blog/2021/08/04/understanding-rpath-with-cmake/) and 
> [here](https://squareys.de/blog/rpath-for-dummies/) 

In addition, we are setting the C++ standard to version 11 using the `CXXFLAGS` environment variable.

Lastly, we tell SCons to compile the given source files to an executable with the name `helloworld`.

If you run this program and everything worked you will receive the following output in the console:
```sh
7
```

## The package manager way
As we now have a better understanding of the different environment variables of the SCons built-system, let's explore
how we can integrate C++ package managers here. For the following examples we will integrate the 
[`fmt` formatting library](https://github.com/fmtlib/fmt) with both vcpkg and conan.

### vcpkg
For using the vcpkg package manager we assume similarly to the last section the following folder structure:

```sh
vcpkg-simple-scons-test/ # root folder name
|
+--src/                   # source code of the program
|   |                     # we are building
|   +--helloworld.cpp
|   +--helloworld.h
|
+--SConstruct             # SCons build file
```

Next up, we will install vcpkg. For that, you will need to follow the steps described in the [vcpkg documentation](https://learn.microsoft.com/en-us/vcpkg/get_started/get-started?pivots=shell-powershell)
until after you have set up the `VCPKG_ROOT` path variable.
If you already have vcpkg installed and set up, you can skip this step.

> [!NOTE]
> If you want to set up your project to use a CI/CD pipeline like GitHub Actions you will want to move the `vcpkg` install
> folder inside your project as a git submodule. In this example we will follow the official docs instead.



### conan

