---
title: "Chapter 3 - Build system"
weight: 4
date: 2021-02-12 12:49:00
description: "Chapter 3 - Ziglang's build system in detail."
---

# Build Modes  

Zig provides four build modes, with debug being the default as it produces the shortest compile times.

|               | Runtime Safety | Optimizations |
|---------------|----------------|---------------|
| Debug         | Yes            | No            |
| ReleaseSafe  | Yes            | Yes, Speed    |
| ReleaseSmall | No             | Yes, Size     |
| ReleaseFast  | No             | Yes, Speed    |

These may be enabled in `zig run` and `zig test` with the arguments `-O ReleaseSafe`, `-O ReleaseSmall` and `-O ReleaseFast`.

Users are recommended to develop their software with runtime safety enabled, despite its small speed disadvantage.

# Outputting an Executable

The commands `zig build-exe`, `zig build-lib`, and `zig build-obj` can be used to output executables, libraries and objects, respectively. These commands take in a source file and arguments.

Some common arguments:
- `-fsingle-threaded`, which asserts the binary is single-threaded. This will turn thread safety measures such as mutexes into no-ops.
- `-fstrip`, which removes debug info from the binary.
- `--dynamic`, which is used in conjunction with `zig build-lib` to output a dynamic/shared library.

Let's create a tiny hello world. Save this as `tiny-hello.zig`, and run `zig build-exe .\tiny-hello.zig -O ReleaseSmall -fstrip -fsingle-threaded`. Currently for `x86_64-windows`, this produces a 2.5KiB executable.

<!--no_test-->
```zig
const std = @import("std");

pub fn main() void {
    std.io.getStdOut().writeAll(
        "Hello World!",
    ) catch unreachable;
}
```

# Cross compilation

By default, Zig will compile for your combination of CPU and OS. This can be overridden by `-target`. Let's compile our tiny hello world to a 64 bit arm linux platform.

`zig build-exe .\tiny-hello.zig -O ReleaseSmall -fstrip -fsingle-threaded -target aarch64-linux`

[QEMU](https://www.qemu.org/) or similar may be used to conveniently test executables made for foreign platforms.

Some CPU architectures that you can cross-compile for:
- `x86_64`
- `arm`
- `aarch64`
- `i386`
- `riscv64`
- `wasm32`

Some operating systems you can cross-compile for:
- `linux`
- `macos`
- `windows`
- `freebsd`
- `netbsd`
- `dragonfly`
- `UEFI`

Many other targets are available for compilation, but aren't as well tested as of now. See [Zig's support table](https://ziglang.org/learn/overview/#wide-range-of-targets-supported) for more information; the list of well tested targets is slowly expanding.

As Zig compiles for your specific CPU by default, these binaries may not run on other computers with slightly different CPU architectures. It may be useful to instead specify a specific baseline CPU model for greater compatibility. Note: choosing an older CPU architecture will bring greater compatibility, but means you also miss out on newer CPU instructions; there is an efficiency/speed versus compatibility trade-off here.

Let's compile a binary for a sandybridge CPU (Intel x86_64, circa 2011), so we can be reasonably sure that someone with an x86_64 CPU can run our binary. Here we can use `native` in place of our CPU or OS, to use our system's.

`zig build-exe .\tiny-hello.zig -target x86_64-native -mcpu sandybridge`

Details on what architectures, OSes, CPUs and ABIs (details on ABIs in the next chapter) are available can be found by running `zig targets`. Note: the output is long, and you may want to pipe it to a file, e.g. `zig targets > targets.json`.

# Zig Build

The `zig build` command allows users to compile based on a `build.zig` file. `zig init-exe` and `zig init-lib` can be used to give you a baseline project.

Let's use `zig init-exe` inside a new folder. This is what you will find.
```
.
├── build.zig
└── src
    └── main.zig
```
`build.zig` contains our build script. The *build runner* will use this `pub fn build` function as its entry point - this is what is executed when you run `zig build`.

<!--no_test-->
```zig
const std = @import("std");

// Although this function looks imperative, note that its job is to
// declaratively construct a build graph that will be executed by an external
// runner.
pub fn build(b: *std.Build) void {
    // Standard target options allows the person running `zig build` to choose
    // what target to build for. Here we do not override the defaults, which
    // means any target is allowed, and the default is native. Other options
    // for restricting supported target set are available.
    const target = b.standardTargetOptions(.{});

    // Standard optimization options allow the person running `zig build` to select
    // between Debug, ReleaseSafe, ReleaseFast, and ReleaseSmall. Here we do not
    // set a preferred release mode, allowing the user to decide how to optimize.
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "tmp",
        // In this case the main source file is merely a path, however, in more
        // complicated build scripts, this could be a generated file.
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    // This declares intent for the executable to be installed into the
    // standard location when the user invokes the "install" step (the default
    // step when running `zig build`).
    b.installArtifact(exe);

    // This *creates* a Run step in the build graph, to be executed when another
    // step is evaluated that depends on it. The next line below will establish
    // such a dependency.
    const run_cmd = b.addRunArtifact(exe);

    // By making the run step depend on the install step, it will be run from the
    // installation directory rather than directly from within the cache directory.
    // This is not necessary, however, if the application depends on other installed
    // files, this ensures they will be present and in the expected location.
    run_cmd.step.dependOn(b.getInstallStep());

    // This allows the user to pass arguments to the application in the build
    // command itself, like this: `zig build run -- arg1 arg2 etc`
    if (b.args) |args| {
        run_cmd.addArgs(args);
    }

    // This creates a build step. It will be visible in the `zig build --help` menu,
    // and can be selected like this: `zig build run`
    // This will evaluate the `run` step rather than the default, which is "install".
    const run_step = b.step("run", "Run the app");
    run_step.dependOn(&run_cmd.step);

    // Creates a step for unit testing. This only builds the test executable
    // but does not run it.
    const unit_tests = b.addTest(.{
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    const run_unit_tests = b.addRunArtifact(unit_tests);

    // Similar to creating the run step earlier, this exposes a `test` step to
    // the `zig build --help` menu, providing a way for the user to request
    // running the unit tests.
    const test_step = b.step("test", "Run unit tests");
    test_step.dependOn(&run_unit_tests.step);
}
```

`main.zig` contains our executable's entry point.

<!--no_test-->
```zig
const std = @import("std");

pub fn main() !void {
    // Prints to stderr (it's a shortcut based on `std.io.getStdErr()`)
    std.debug.print("All your {s} are belong to us.\n", .{"codebase"});

    // stdout is for the actual output of your application, for example if you
    // are implementing gzip, then only the compressed bytes should be sent to
    // stdout, not any debugging messages.
    const stdout_file = std.io.getStdOut().writer();
    var bw = std.io.bufferedWriter(stdout_file);
    const stdout = bw.writer();

    try stdout.print("Run `zig build test` to run the tests.\n", .{});

    try bw.flush(); // don't forget to flush!
}

test "simple test" {
    var list = std.ArrayList(i32).init(std.testing.allocator);
    defer list.deinit(); // try commenting this out and see if zig detects the memory leak!
    try list.append(42);
    try std.testing.expectEqual(@as(i32, 42), list.pop());
}
```

Upon using the `zig build` command, the executable will appear in the install path. Here we have not specified an install path, so the executable will be saved in `./zig-out/bin`.

# Builder

Zig's [`std.Build`](https://ziglang.org/documentation/master/std/#A;std:Build) type contains the information used by the build runner. This includes information such as:

- the build target
- the release mode
- locations of libraries
- the install path
- build steps

# CompileStep

The `std.build.CompileStep` type contains information required to build a library, executable, object, or test.

Let's make use of our `Builder` and create a `CompileStep` using `Builder.addExecutable`, which takes in a name and a path to the root of the source.

<!--no_test-->
```zig
const Builder = @import("std").build.Builder;

pub fn build(b: *Builder) void {
    const exe = b.addExecutable(.{
        .name = "init-exe",
        .root_source_file = .{ .path = "src/main.zig" },
    });
    b.installArtifact(exe);
}
```

# Modules

The Zig build system has the concept of modules, which are other source files written in Zig. Let's make use of a module.

From a new folder, run the following commands.
```
zig init-exe
mkdir libs
cd libs
git clone https://github.com/Sobeston/table-helper.git
```

Your directory structure should be as follows.

```
.
├── build.zig
├── libs
│   └── table-helper
│       ├── example-test.zig
│       ├── README.md
│       ├── table-helper.zig
│       └── zig.mod
└── src
    └── main.zig
```

To your newly made `build.zig`, add the following lines.

<!--no_test-->
```zig
    const table_helper = b.addModule("table-helper", .{
        .source_file = .{ .path = "libs/table-helper/table-helper.zig" }
    });
    exe.addModule("table-helper", table_helper);
```

Now when run via `zig build`, [`@import`](https://ziglang.org/documentation/master/#import) inside your `main.zig` will work with the string "table-helper". This means that main has the table-helper package. Packages (type [`std.build.Pkg`](https://ziglang.org/documentation/master/std/#std;build.Pkg)) also have a field for dependencies of type `?[]const Pkg`, which is defaulted to null. This allows you to have packages which rely on other packages. 

Place the following inside your `main.zig` and run `zig build run`. 

<!--no_test-->
```zig
const std = @import("std");
const Table = @import("table-helper").Table;

pub fn main() !void {
    try std.io.getStdOut().writer().print("{}\n", .{
        Table(&[_][]const u8{ "Version", "Date" }){
            .data = &[_][2][]const u8{
                .{ "0.7.1", "2020-12-13" },
                .{ "0.7.0", "2020-11-08" },
                .{ "0.6.0", "2020-04-13" },
                .{ "0.5.0", "2019-09-30" },
            },
        },
    });
}
```

This should print this table to your console.

```
Version Date       
------- ---------- 
0.7.1   2020-12-13 
0.7.0   2020-11-08 
0.6.0   2020-04-13 
0.5.0   2019-09-30 
```


Zig does not yet have an official package manager. Some unofficial experimental package managers however do exist, namely [gyro](https://github.com/mattnite/gyro) and [zigmod](https://github.com/nektro/zigmod). The `table-helper` package is designed to support both of them.

Some good places to find packages include: [astrolabe.pm](https://astrolabe.pm), [zpm](https://zpm.random-projects.net/), [awesome-zig](https://github.com/nrdmn/awesome-zig/), and the [zig tag on GitHub](https://github.com/topics/zig).

# Build steps

Build steps are a way of providing tasks for the build runner to  execute. Let's create a build step, and make it the default. When you run `zig build` this will output `Hello!`. 

<!--no_test-->
```zig
const std = @import("std");

pub fn build(b: *std.build.Builder) void {
    const step = b.step("task", "do something");
    step.makeFn = myTask;
    b.default_step = step;
}

fn myTask(self: *std.build.Step, progress: *std.Progress.Node) !void {
    std.debug.print("Hello!\n", .{});
    _ = progress;
    _ = self;
}
```

We called `b.installArtifact(exe)` earlier - this adds a build step which tells the builder to build the executable.

# Generating Documentation

The Zig compiler comes with automatic documentation generation. This can be invoked by adding `-femit-docs` to your `zig build-{exe, lib, obj}` or `zig run` command. This documentation is saved into `./docs`, as a small static website.

Zig's documentation generation makes use of *doc comments* which are similar to comments, using `///` instead of `//`, and preceding globals.

Here we will save this as `x.zig` and build documentation for it with `zig build-lib -femit-docs x.zig -target native-windows`. There are some things to take away here:
-  Only things that are public with a doc comment will appear
-  Blank doc comments may be used
-  Doc comments can make use of subset of markdown
-  Things will only appear inside generated documentation if the compiler analyses them; you may need to force analysis to happen to get things to appear.

<!--no_test-->
```zig
const std = @import("std");
const w = std.os.windows;

///**Opens a process**, giving you a handle to it. 
///[MSDN](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess)
pub extern "kernel32" fn OpenProcess(
    ///[The desired process access rights](https://docs.microsoft.com/en-us/windows/win32/procthread/process-security-and-access-rights)
    dwDesiredAccess: w.DWORD,
    ///
    bInheritHandle: w.BOOL,
    dwProcessId: w.DWORD,
) callconv(w.WINAPI) ?w.HANDLE;

///spreadsheet position
pub const Pos = struct{
    ///row
    x: u32,
    ///column
    y: u32,
};

pub const message = "hello!";

//used to force analysis, as these things aren't otherwise referenced.
comptime {
    _ = OpenProcess;
    _ = Pos;
    _ = message;
}

//Alternate method to force analysis of everything automatically, but only in a test build:
test "Force analysis" {
    comptime {
        std.testing.refAllDecls(@This());
    }
}
```

When using a `build.zig` this may be invoked by setting the `emit_docs` field to `.emit` on a `CompileStep`. We can create a build step to generate docs as follows and invoke it with `$ zig build docs`.

<!--no_test-->
```zig
const std = @import("std");

pub fn build(b: *std.build.Builder) void {
    const mode = b.standardReleaseOptions();

    const lib = b.addStaticLibrary("x", "src/x.zig");
    lib.setBuildMode(mode);
    lib.install();

    const tests = b.addTest("src/x.zig");
    tests.setBuildMode(mode);

    const test_step = b.step("test", "Run library tests");
    test_step.dependOn(&tests.step);

    //Build step to generate docs:
    const docs = b.addTest("src/x.zig");
    docs.setBuildMode(mode);
    docs.emit_docs = .emit;
    
    const docs_step = b.step("docs", "Generate docs");
    docs_step.dependOn(&docs.step);
}
```

This generation is experimental, and often fails with complex examples. This is used by the [standard library documentation](https://ziglang.org/documentation/master/std/).

When merging error sets, the left-most error set's documentation strings take priority over the right. In this case, the doc comment for `C.PathNotFound` is the doc comment provided in `A`.

<!--no_test-->
```zig
const A = error{
    NotDir,

    /// A doc comment
    PathNotFound,
};
const B = error{
    OutOfMemory,

    /// B doc comment
    PathNotFound,
};

const C = A || B;
```

# End of Chapter 3

This chapter is incomplete. In the future it will contain advanced usage of `zig build`.

Feedback and PRs are welcome.
