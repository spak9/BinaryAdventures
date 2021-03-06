# Part 1 - Basic Function Instrumentation

Whenever you're reverse engineering a binary, you're typically trying to 
achieve two things:
1. Understanding functionality, that is, how does the binary work and at a finer-grained level, 
what are specific functions doing. This includes understanding the inputs to a function,
the internal functionality of the function, and the return of the function, if any.
2. Understanding data, that is, what are the important pieces of data that is 
scattered around the binary and what are its uses. For example, extern/global data.


In this part 1, I want to discuss my thoughts on how I typically tackle this
first component of "understanding functionality". This usually means having 
a target function in mind and trying to `instrument` it, that is,
inserting code at X point within an existing binary to observe or modify 
the existing code in some way. Thankfully `frida` does the actual 
instrumentation via their `frida gum` APIs, but we as analysts get to 
take advantage of that via their Javascript APIs.


For basic interception of calls to X function, we'll need to use the 
`Interceptor` object and its API.
The main function we'll be using is:
`Interceptor.attach(target, callbacks[, data])`,
where `target` is a `NativePointer` and `callbacks` is an object that contains various
other objects, two of which are called: 
1. `onEnter(args)`, which is a callback that is executed whenever our `target` function is executed, 
with `args` being an array of `NativePointer`s, a point that is often overlooked and something we'll
revisit soon.
2. `onLeave(retval)`, which is also a callback that is executed whenever `target` function is completed,
with `retval` being a `NativePointer` of the return value, if any.


Firstly, we need a `NativePointer` that can be our target for hooking, but how exactly do we get that?
A `NativePointer` is exactly what it sounds like, it's a pointer class, in which an instance is a 
variable whose value is a memory address. That means we need a `NativePointer` whose value is
the memory address of the function we'd like to hook! In order to find that memory address, we need to 
learn about the `Module` API.

I like to think of the `Module` object as a object representing a 
typical shared object/executable (from the ELF perspective). 
There are 2 main ways I like to get hold of `Module` objects:
1. `Process.enumerateModules()` which returns an array of `Module` objects
that are loaded in the process. Bascially all the shared objects a process is using.
2. `Process.findModuleByName/Address(name_or_address)` which returns a
`Module` instance or null if not available. 
`Module`s have attributes like `name`, `size(bytes)`, and most importantly,
the `address`, which is a `NativePointer` of the base address of the module.
For example, the following commands will exemplify the process of finding 
the `Module` you're interested in.
```
frida a.out
<frida header commandline stuff...>
[Local::a.out ]-> Process.findModuleByName("a.out")
{
    "base": "0x55890b0000",
    "name": "a.out",
    "path": "/home/steven/Desktop/code/BinaryAdventures/frida/part_1_basic_arm64_hooking/a.out",
    "size": 73728
}
```

`Module` objects have a few important functions for finding important
information within the executable itself.
1. `Module.enumerateImports/Exports/Symbols()` - returns an array of 
the specified data objects from the Module, all of which contain an 
`address` field. This can then tell us external 
functions/variables and other symbols (if not stripped) and their addresses 
in memory, in which we can then hook into! 

Coming full circle, you examine the `Module`, which the typically the actual executable
or some type of shared object you're interested,
then attempt to find the symbol/export's address (NativePointer). We
can then pass that into `Interceptor.attach(target, ..)`.

## Example Code

