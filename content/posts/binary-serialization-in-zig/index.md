---
title: How to Serialize Data in Zig
author: Joey Reed
date: 2025-01-11
draft: false
summary:    
tags: ["zig"]
ShowToc: true
---

A lot of my work involves storing and processing large amounts of signal data from sensor systems.  I have two options - store the data in text files or binary files.  Text files are human readable, but they're too big and slow to work with.  Binary files are my only practical option.  As a result, any programming language I use must provide a simple way to serialize and deserialize data structures to binary files.  I'm going to explain how to do this in [Zig](https://ziglang.org) - a modern systems programming language.

## Handling Binary Files in C

Call me old fashioned, but I really like the way C handles binary serialization.  Let's say you want to write an array to a binary file.  No problem.

```c
fwrite(array, sizeof(array[0]), sizeof(array)/sizeof(array[0]), fid);
```

What about a struct?  Piece of cake.

```c
fwrite(&struct_instance, sizeof(MyStruct_t), 1, fid);
```
  
In C, it's extremely easy to convert between different types of pointers.  And if you don't want to do the cast yourself, the compiler can implicity do it for you.  It'll even convert an array, to a pointer, to it's first element.  Otherwise the example with the array wouldn't work.  The compiler assumes you know what you're doing and gets out of the way.  I respect that.  

## Pointers in Zig

Zig, like many other modern systems languages, discourages you from using raw pointers.  At least that's the impression I get when reading over the documentation.  Don't
get me wrong, it still has raw pointers.  It just tries really hard to get you to use other data structures like arrays, many-item pointers, and slices.  

Arrays and pointers are not interchangeable like they are in C.  Case in point, the Zig compiler won't let you pass an array to a function that expects a pointer.  Implicit type conversion (aka coercion) does happen, but the rules are stricter than they are in C.  It's worth taking a look at the coercion rules in the documentation:

* [https://ziglang.org/documentation/0.13.0/#Type-Coercion](https://ziglang.org/documentation/0.13.0/#Type-Coercion)

Many-item pointers are advertised as being like C pointers.  They point to memory, but don't say anything about how big the piece of memory is.  If you want a memory address and a size,  use a slice.  Zig really likes slices.  And for good reason.  It gives the compiler a way to enforce bounds checking.            

C doesn't have any builtin bounds checking.  If you want it, you've got to implement it yourself.  Take a look at those C code snippets again.  Together, the second and third function arguments determine how many bytes of data past the pointer you want to write.  You can put any non-negative numbers in there.  The compiler doesn't check whether it makes sense of not.  It's up to you.  Depending only your application, this can lead to some seriously buggy code.  


## Array Serialization Example

Ok, let's get down to business.  One of things I do all the time at work is write arrays of floating point values to binary files.  Zig has a function in its standard library for this called `write`.  The problem is that it only writes byte slices.  So the first thing I had to figure out was how to convert arrays or slices of floats to slices of bytes.  I got a decent solution that worked for floats, but realized I could use Zig's `comptime` feature to generalize it to any slice.  Here's what I settled on:         

```zig 
fn transmuteSlice(comptime T: type, x: []T) []u8 { 
    const num_bytes = @sizeOf(T) * x.len;
    const x0: [*]u8 = @ptrCast(x);
    const x1: [ ]u8 = x0[0..num_bytes];
    return x1;
}
``` 
Even if you don't know Zig, you can probably piece this together.  Here are a few remarks just in case:

* `comptime T: type` -  The first function argument must be a type known at compile time.
* `@sizeOf(T)` - compiler builtin that returns the size of the compile time type `T`, in bytes.
* `@ptrCast(x)` - compiler builtin that converts `x`'s internal many-item pointer `[*]T` to a `[*]u8`.
* `x0[0..num_bytes]` - turns the many-item pointer into a slice. 
 
I stole the name "transmute" from Odin ([https://odin-lang.org/](https://odin-lang.org/)), another powerful systems programming language.  It captures exactly what
the function is doing:

> **transmute**: to change or alter in form, appearance, or nature. 

`transmuteSlice` isn't returning creating a new slice, it's reinterpreting the memory in `[]T` as a `[]u8`.  Think of it as the slice equivalent of pointer casting.

Let's see it in action.  Here's a program that does what I described at the beginning: write chunks of floating point values to a binary file.  To verify that the file writing worked correctly, I read the data back in from the file and compared it to the original "parent" array I used for the writes.  Zig's `read` standard library implementation is like `write`, it only accepts byte slices.  

```zig {linenos=true}

// 1. Import the Standard Library and the different name spaces we'll use
const std    = @import("std");
const fs     = std.fs;
const print  = std.debug.print;
const mem    = std.mem;
const assert = std.debug.assert;


// 2. Define the "main" function
pub fn main() !void {

    // 3. Initialize the test array
    var x0: [80] f32 = undefined;    
    for (0..x0.len) | i | { 
        x0[i] = @floatFromInt(i); 
    }
     
    // 4. Create a new directory and a file in the directory
    try fs.cwd().makePath("TEST");
    var fid = try fs.cwd().createFile("TEST/data.bin", .{});
    
    // 5. Write the data to the file and close
    var start : u32 = 0;
    for (0..10) |_| {
        _ = try fid.write(transmuteSlice(f32, x0[start..start+8]));
        start += 8;
    }
    fid.close(); 
    
    // 6. Open the file for reading
    fid = try fs.cwd().openFile("TEST/data.bin", .{ .mode = .read_only });

    // 7. Read the data from the file in chunks and store in new array
    var y0: [80] f32 = undefined;
    start = 0;
    for (0..10) |_| {
        _ = try fid.read(transmuteSlice(f32, y0[start..start+8]));
        start += 8;
    }
    fid.close();

    
    // 8. Check that the slices have the same values
    assert(mem.eql(f32, &x0, &y0));

    
    // 9. Remove the directory
    try fs.cwd().deleteTree("TEST");
}
```

This is not an introductory Zig tutorial, so I'm not going to explain this program in gory detail.  To be honest, I bet you can figure out every line without my commentary - Zig is very readable.  Hopefully the program comments should help a little bit too.  I did want to point out a few things.

* **Line 21** - The second argument to `createFile` is a struct with default values.  Even though the default values are what we want, we have to pass something in.  In this case, an "anonymous struct literal" is what you want.
* **Line 26,28** - This is how you invoke `transmuteSlice`.
* **Line 45** - The second and third arguments to `mem.eql` should be slices.  We're passing in pointers to arrays: `*[80]f32`.  What gives?  Pointers to arrays coerce to slices!
 
## Struct Serialization

After I figured out how to transmute slices to byte slices, I started wondering about structure serialization.  How could I turn a struct I define into a byte slice?  One option is to instantiate the struct, put it in a slice, and just use the `transmuteSlice` function.  Meh.  Too much work.  Here's a better solution:      

```zig
fn transmutePtr(comptime T: type, x: *T) []u8 {  
    const num_bytes = @sizeOf(T);
    const x0: [*]u8 = @ptrCast(x);
    const x1: [ ]u8 = x0[0..num_bytes];
    return x1;
}
```

and here's how you's use it:

```zig  
const MyStruct = struct {
    a: u8,
    b: u8
};

const x = MyStruct {.a=1, .b=3};
const y = transmutePtr(MyStruct, @constCast(&x));

```

I'm pretty happy with that.  

## Conclusion

The purpose of this post was to develop a simple way to serialize arrays and data structures in Zig.  I succeeded.  My conclusion - Zig is a great language and I'm going to contunue developing in it.     
