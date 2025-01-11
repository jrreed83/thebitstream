---
title: How to Serialize Data in Zig
author: Joey Reed
date: 2025-01-01
draft: true
summary:  
tags: ["zig"]
ShowToc: true
---

A lot of my work involves storing and processing large amounts of digital samples from data acquisition (DAQ) systems.  I have two options - store the data in text files or binary files.  Text files have the advantage of being human readable, but they're too big and slow to work with.  Binary files are my only practical option.  As a result, any programming language I use must provide a simple way to serialize and deserialize arrays, and other data structures, to binary files.  In this post, I'll show how to do this in [Zig](https://ziglang.org).

## Handling Binary Files in C

Call me old fashioned, but I really like the way C handles binary serialization.  Let's say you want to write an array called to a binary file.  No problem.

```c
fwrite(array, sizeof(array[0]), sizeof(array)/sizeof(array[0]), fid);
```

What about a struct?  Piece of cake.

```c
fwrite(&struct_instance, sizeof(MyStruct_t), 1, fid);
```
  
In C, it's extremely easy to convert between different types of pointers.  And if you don't want to do the cast yourself, the compiler implicity does it for you.  It'll even convert an array to a pointer to it's first element.  Otherwise the example with the array wouldn't work.  It assumes you know what you're doing and gets out of the way.  I respect that.  

## Pointers in Zig

Zig, like many other modern systems languages, discourages you from using raw pointers.  At least that's the impression I get when looking at the documentation.  Don't
get me wrong, the language still has raw pointers.  It just tries really hard to get you to use other data structures like arrays, many-item pointers, and slices.  

Arrays and pointers are not interchangeable like they are in C.  Case in point, the Zig compiler won't let you pass an array to a function that expects a pointer.  Implicit type conversion (aka coercion) does happen, but the rules are stricter than they are in C.  It's worth taking a look at the coercion rules in the documentation:

* [https://ziglang.org/documentation/0.13.0/#Type-Coercion](https://ziglang.org/documentation/0.13.0/#Type-Coercion)

Many-item pointers are advertised as being like C pointers.  They point to memory, but don't say anything about how big the piece of memory is.  If you want a memory address and a size,  use a slice.  Zig really likes slices.  Take a look at the standard library documentation; slice declarations everywhere.  And for good reason.  It gives the compiler a way to enforce bounds checking.            

C doesn't have any builtin bounds checking.  If you want it, you've got to implement it yourself.  Take a look at those C code snippets again.  Together, the second and third function arguments determine how many bytes of data past the pointer you want to write.  You can put any non-negative numbers in there.  The compiler doesn't check whether it makes sense of not.  It's up to you.  Depending only your application, this can lead to some seriously buggy code.  


## Array Serialization Example

Ok, let's get down to business.  One of things I do all the time at work is write arrays of floating point values to binary files.  Zig has a functions in its standard library for this called `write`.  The problem is that it only writes byte slices.  The first thing I had to figure out is how to convert arrays or slices of floats to slices of bytes.  Here's what I settled on:         

```zig 
fn transmuteSlice(comptime T: type, x: []T) []u8 {
 
    const num_bytes = @sizeOf(T) * x.len;
    const x0: [*]u8 = @ptrCast(x);
    const x1: [ ]u8 = x0[0..num_bytes];
 
    return x1;
}
``` 
* `comptime` is short for compile time - the first function argument must be a type known at compile time.
* `@sizeOf(T)` returns the size of the compile time type `T`, in bytes.
* `@ptrCast(x)` converts `x`'s internal many-item pointer `[*]T` to a `[*]u8`.
* `x0[0..num_bytes]` turns the many-item pointer into a slice. 
 
I stole the name "transmute" from Odin ([https://odin-lang.org/](https://odin-lang.org/)), another powerful systems programming language.  It captures exactly what
the function is doing:

> **transmute**: to change or alter in form, appearance, or nature. 

`transmuteSlice` isn't returning creating a new slice, it's reinterpreting the memory in `[]T` as a `[]u8`.  Think of it as the slice equivalent of pointer casting.

```zig

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

```



### Struct Serialization

```zig
fn transmutePtr(comptime T: type, x: *T) []u8 {
  
      const num_bytes = @sizeOf(T);
      const x0: [*]u8 = @ptrCast(x);
      const x1: [ ]u8 = x0[0..num_bytes];
  
      return x1;
}

``` 
## Conclusion


