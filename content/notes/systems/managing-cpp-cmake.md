+++
title = "Managing C++ Projects Using CMake"
+++

> From [studyplan.dev](https://www.studyplan.dev/cmake/cpp-build-process)

This course is about how to use **CMake** to build C++ projects.

We learned C++ in my degree, but never how to use **CMake** (maybe because my already credited courses from my previous DEC talked about it). I actually used it a bit in an internship to build a program running on multiple ESP32s with the ESP-IDF.

I want to use it more in my next projects, be it in C or C++.

## Lesson 1 - The C++ Build Process

> An introduction to the build pipeline, and how we can orchestrate it manually.

### Chapter 1 - The C++ Build Process
This chapter was mostly revision of what I already knew about compilers, with added information about compiler flags and the `ar` program for static libraries.

It presents mostly each step of the compilation process (preprocessing, compiling, assembling, linking) and shows the different tools needed to do it by hand in the CLI before using **CMake**.

#### 📌 Learned 
- Compiler flags for different steps of the compilation process
- `ar` for archiving static libraries
- Position-Independent Code for dynamic libraries
- A simple Hello World program with `<iostream>` is 25k lines after preprocessing since it includes the whole iostream headers

---
### Chapter 2 - Project Structure and Dependency Management
This chapter presents the organization of a C++ project as well as build systems and their limitations for complex projects. It talked about versioning, project layout, API vs ABI, etc.

#### 📌 Learned 
- Typical C++ projet layout consisting of `src` folder for the source code and private headers, `include` folder for the public headers and`libs` folder for external libraries
- Two ways of separating multiple parts of a project like `core` and `graphics` by either having folder for each part under `src` and `include` or having `src` and `include` in each folder of each part
- Compiler flags for linking libraries (`-I`, `-L`, `-l`)
- Semantic versioning
- Platform differences such has two file system for a `dll` on **Windows** (`.lib` and `.dll`) vs `.so` on **Linux** and **macOS**
- Limitations of Makefiles and IDE Project files such as vendor lock-in, dependency on GUI or the monolithic unreadable single **Makefile** and duplication when splitting it

---
### Chapter 3 - Introducing **CMake**
As said before, I had already used **CMake** in an intership before and was already a familiar with the basics. I therefore did not learn much about the basic commands like `add_executable`, `add_library`, `target_link_libraries`, etc.

I did learn more reasons as to why use **CMake** and little tips about structure and versions.

#### 📌 Learned 
- Why **CMake**? *portability*, *reproducibility* and *consistency*, and *maintainability* and *scalability*
- **CMake** alternatives such as **Bazel** or **Meson**, but **CMake** remains the most widely adopted
- Tips on choosing a version of **CMake**: 
	- Old enough to be compatible with most tools
	- New enough to have modern features and still be supported
- Tips about choosing the `target_` version of a command: *granularity*, *clarity* and *transitivity*

> [!tip] `compile_commands` 
> To generate a `compile_commands.json` file to be used by the `clangd` LSP for ease of programming (auto-completion, linting, etc.), add 
> ```cmake
> set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
> ```
