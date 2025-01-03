---
title: "How to use the scons build-system with C++ software package managers (vcpkg, conan)"
date: 2024-12-29T18:00:05+01:00
draft: true
---

## Introduction
As you know from perhaps my previous articles on [how to contribute to Godot](https://paddy-exe.github.io/posts/contributing-to-godot-visual-shader-nodes/),
[creating your own GDExtension](https://paddy-exe.github.io/posts/gdextension-how-to-write-your-first-extension/) or from the official Godot docs,
Godot uses the [scons build-system](https://scons.org/) to compile the engine code written in C++ to an executable.
Scons tells the compiler specifically HOW to build together that executable.
Another build-system you will have probably heard of is the widely popular and de-facto industry standard CMake.

These build-systems help you not only to compile your own code but also to combine your code together with C++ code others have written as libraries.
These libraries also get updates so if you use several libraries in your projects it can get quite tedious managing everything.
Here come software package manager to your rescue. They let you "install" a library into your project (meaning they download that source code and compile it)
and then hold a record of which libraries with what version you have "installed" in your project.

The best known software package manager for C and C++ are [vcpkg](https://vcpkg.io/en/) and [conan](https://conan.io/).
Both have a lot of features in common and are different in more advanced use-cases in which I won't go into here.
Both have excellent support for CMake since - again - it is the de-facto standard. However, how do you use them with the scons build-system?

## The manual way
To understand how scons can generally be used with third-party libraries that can be integrated via git-submodules as well we will first explore that.
This will later help to understand how it can be done for vcpkg as well.



