+++
title = "Learning Zig"
+++

## [Overview of features](https://ziglang.org/learn/overview/)
### Control Flow
Zig has no **hidden control flow** or any other hidden allocations and no preprocessor like C nor macros. They give a few examples, like 
- decorators like `@property` from D that are called when the field is accessed;
- function overloading from C++ and Rust;
- exceptions throwing and catching that might prevent other functions from being called.
All of  these are functions that are not explicitly called and can cause unwanted side effects, while Zig promotes code maintenance and readability.

### Build mode
Zig's build mode let's you decide between two parameters: optimization and safety. With the combination of those two, you can have the following build modes: `Debug` (no optimizations), `ReleaseSafe` (optimizations and safety checks), `ReleaseFast` (no safety checks) and `ReleaseSmall` (size optimizations and no safety checks). This permits targeting different plateforms and goals.

### C improvements
Zig also claims a couple improvements over C. I'll have to test them later.
- One compilation unit optimized together.
- Standard library provides essential data structures such as hash maps and array lists.
- Advanced CPU features enabled by default.
- Standard library integrated with `libc`, but does not depend on it, whatever that means (?)

### Declaration order
Top level declarations are order-independent. That should prove useful for function declarations, but not for specifically for variables. I find it will make them harder to read.
```zig
var y: i32 = add(10, x);
const x: i32 = add(12, 34);

fn add(a: i32, b: i32) i32 {
    return a + b;
}
```

### Optional type, no null pointer
No null pointers (pointing to 0), but an optional type at type declaration by prefixing with `?`, which can then be compared with `null`:
```zig
const ptr: ?*i32 = @ptrFromInt(0x0);
assert(ptr == null);
```
It can also be unwrapped with `orelse`, `if` and `while`.

### Manual memory management
A Zig program can be used anywhere:
- Desktop applications
- Low-latency servers
- Databases
- Operating System kernels
- Embedded devices
- Real-time software, e.g. live performances, airplanes, pacemakers
- In web browsers or other plugins with WebAssembly
- By other programming languages, using the C ABI
Any functions that need to allocate memory accept an allocator parameter. Zig allows gives us `defer` and `errdefer`, two keywords that lets you write lines of code needed to be run at the end of the function wherever in it as to not forget, for example, to deallocate memory by putting the deallocation next to the allocation.

### C ABI, libraries and `libc`
Zig lets you export code a library with the C ABI for other languages to use with the `export` keyword. Then you can make a static library with `zig build-lib code.zig` and a shared one with `zig build-lib code.zig -dynamic`.

One top of that, Zig ships with `libc` for multiple targets and supports building for those regardless of the host.

### Build system and package manager
Zig comes with a build system, rendering my [[managing-cpp-cmake|learning of CMake]] pretty useless. It also seems to come with a package manager (?) by using `zig fetch url`.
