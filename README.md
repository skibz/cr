# cr.h
 
A single file header-only live reload solution for C, written in C++:

- simple public API, 3 functions only to use (and only one to export);
- works and tested on Linux and Windows;
- automatic crash protection;
- automatic static state transfer;
- based on dynamic reloadable binary (.so/.dll);
- MIT licensed;

### Build Status:

Platform | Build Status
-------- | ------------
Linux | [![Build Status](https://travis-ci.org/fungos/cr.svg?branch=master)](https://travis-ci.org/fungos/cr)
Windows | [![Build status](https://ci.appveyor.com/api/projects/status/jf0dq97w9b7b5ihi?svg=true)](https://ci.appveyor.com/project/fungos/cr)

Note that the only file that matters is `cr.h`.

This file contains the documentation in markdown, the license, the implementation and the public api.
All other files in this repository are supporting files and can be safely ignored.

### Example

A (thin) host application executable will make use of `cr` to manage
live-reloading of the real application in the form of dynamic loadable binary, a host would be something like:

```c
#define CR_HOST // required in the host only and before including cr.h
#include "../cr.h"

int main(int argc, char *argv[]) {
    // the host application should initalize a plugin with a context, a plugin
    cr_plugin ctx;

    // the full path to the live-reloadable application
    cr_plugin_load(ctx, "c:/path/to/build/game.dll");

    // call the update function at any frequency matters to you, this will give the real application a chance to run
    while (!cr_plugin_update(ctx)) {
        // do anything you need to do on host side (ie. windowing and input stuff?)
    }

    // at the end do not forget to cleanup the plugin context
    cr_plugin_close(ctx);
    return 0;
}
```

While the guest (real application), would be like:

```c
CR_EXPORT int cr_main(struct cr_plugin *ctx, enum cr_op operation) {
    assert(ctx);
    switch (operation) {
        case CR_LOAD:   return on_load(...);
        case CR_UNLOAD: return on_unload(...);
    }
    // CR_STEP
    return on_update(...);
}
```

### Samples

Two simple samples can be found in the `samples` directory.

The first is one is a simple console application that demonstrate some basic static
states working between instances and basic crash handling tests. Print to output
is used to show what is happening.

The second one demonstrates how to live-reload an opengl application using
 [Dear ImGui](https://github.com/ocornut/imgui). Some state lives in the host
 side while most of the code is in the guest side.

 ![imgui sample](https://i.imgur.com/Nq6s0GP.gif)

#### Running Samples and Tests

The samples and tests uses the [fips build system](https://github.com/floooh/fips). It requires Python and CMake.

```
$ ./fips build            # will generate and build all artifacts
$ ./fips run crTest       # To run tests
$ ./fips run imgui_host   # To run imgui sample
# open a new console, then modify imgui_guest.cpp
$ ./fips make imgui_guest # to build and force imgui sample live reload
```

### Documentation

#### `int (*cr_main)(struct cr_plugin *ctx, enum cr_op operation)`

This is the function pointer to the dynamic loadable binary entry point function.

Arguments

- `ctx` pointer to a context that will be passed from `host` to the `guest` containing valuable information about the current loaded version, failure reason and user data. For more info see `cr_plugin`.
- `operation` which operation is being executed, see `cr_op`.

Return

- A negative value indicating an error, forcing a rollback to happen and failure
 being set to `CR_USER`. 0 or a positive value that will be passed to the
  `host` process.

#### `bool cr_plugin_load(cr_plugin &ctx, const char *fullpath)`

Loads and initialize the plugin.

Arguments

- `ctx` a context that will manage the plugin internal data and user data.
- `fullpath` full path with filename to the loadable binary for the plugin or
 `NULL`.

Return

- `true` in case of success, `false` otherwise.

#### `int cr_plugin_update(cr_plugin &ctx)`

This function will call the plugin `cr_main` function. It should be called as
 frequently as the core logic/application needs.

Arguments

- `ctx` the current plugin context data.

Return

- -1 if a failure happened during an update;
- -2 if a failure happened during a load or unload;
- anything else is returned directly from the plugin `cr_main`.

#### `void cr_plugin_close(cr_plugin &ctx)`

Cleanup internal states once the plugin is not required anymore.

Arguments

- `ctx` the current plugin context data.

#### `cr_op`

Enum indicating the kind of step that is being executed by the `host`:

- `CR_LOAD` A load is being executed, can be used to restore any saved internal
 state;
- `CR_STEP` An application update, this is the normal and most frequent operation;
- `CR_UNLOAD` An unload will be executed, giving the application one chance to
 store any required data.
- `CR_CLOSE` Like `CR_UNLOAD` but no `CR_LOAD` should be expected;

#### `cr_plugin`

The plugin instance context struct.

- `p` opaque pointer for internal cr data;
- `userdata` may be used by the user to pass information between reloads;
- `version` incremetal number for each succeded reload, starting at 1 for the
 first load;
- `failure` used by the crash protection system, will hold the last failure error
 code that caused a rollback. See `cr_failure` for more info on possible values;

#### `cr_failure`

If a crash in the loadable binary happens, the crash handler will indicate the
 reason of the crash with one of these:

- `CR_NONE` No error;
- `CR_SEGFAULT` Segmentation fault. `SIGSEGV` on Linux or
 `EXCEPTION_ACCESS_VIOLATION` on Windows;
- `CR_ILLEGAL` In case of illegal instruction. `SIGILL` on Linux or
 `EXCEPTION_ILLEGAL_INSTRUCTION` on Windows;
- `CR_ABORT` Abort, `SIGBRT` on Linux, not used on Windows;
- `CR_MISALIGN` Bus error, `SIGBUS` on Linux or `EXCEPTION_DATATYPE_MISALIGNMENT`
 on Windows;
- `CR_BOUNDS` Is `EXCEPTION_ARRAY_BOUNDS_EXCEEDED`, Windows only;
- `CR_STACKOVERFLOW` Is `EXCEPTION_STACK_OVERFLOW`, Windows only;
- `CR_STATE_INVALIDATED` Static `CR_STATE` management safety failure;
- `CR_OTHER` Other signal, Linux only;
- `CR_USER` User error (for negative values returned from `cr_main`);

#### `CR_HOST` define

This define should be used before including the `cr.h` in the `host`, if `CR_HOST`
 is not defined, `cr.h` will work as a public API header file to be used in the
  `guest` implementation.

Optionally `CR_HOST` may also be defined to one of the following values as a way
 to configure the `safety` operation mode for automatic static state management
  (`CR_STATE`):

- `CR_SAFEST` Will validate address and size of the state data sections during
 reloads, if anything changes the load will rollback;
- `CR_SAFE` Will validate only the size of the state section, this mean that the
 address of the statics may change (and it is best to avoid holding any pointer
  to static stuff);
- `CR_UNSAFE` Will validate nothing but that the size of section fits, may not
 be necessarelly exact (growing is acceptable but shrinking isn't), this is the
 default behavior;
- `CR_DISABLE` Completely disable automatic static state management;

#### `CR_STATE` macro

Used to tag a global or local static variable to be saved and restored during a reload.

Usage

`static bool CR_STATE bInitialized = false;`


### FAQ / Troubleshooting

#### Q: Why?

A: Read about why I made this [here](https://fungos.github.io/blog/2017/11/20/cr.h-a-simple-c-hot-reload-header-only-library/).

#### Q: My application asserts/crash when freeing heap data allocated inside the dll, what is happening?

A: Make sure both your application host and your dll are using the dynamic
 run-time (/MD or /MDd) as any data allocated in the heap must be freed with
  the same allocator instance, by sharing the run-time between guest and
   host you will guarantee the same allocator is being used.

#### Q: Can we load multiples plugins at the same time?

A: Yes. This should work without issues on Windows. On Linux, there may be 
issues with signal handling with the crash protection as it does not have the
plugin context to know which one crashed. This should be fixed in a near future.

### License

The MIT License (MIT)

Copyright (c) 2017 Danny Angelo Carminati Grein

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
