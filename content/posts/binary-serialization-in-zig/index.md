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

## Handling Binary Files in Zig

Call me old fashioned, but I really like the way C handles data serialization.  Let's say you want to write an array to a binary file.

```c
fwrite(x, sizeof(x[0]), sizeof(x)/sizeof(x[0]), fid);
```

How about a struct?  Piece of cake.

```c
fwrite(&x, sizeof(MyStruct_t), 1, fid);
```
  
In C, casting between different pointer types is easy.  And if you don't want to do the cast yourself, the compiler implicity does it for you. It 
assumes you know what you're doing and gets out of the way.  I respect that.  

Zig, like many other modern systems languages, tends to shy away from using raw pointers.  At least that's the impression I get when looking at the documentation.  Don't
get me wrong, the language still has raw pointers.  It just tries really hard to get you to use its *slice* data structure.   A slice is just a pointer and a length; it 
helps the compiler help you with bounds checking.  

C doesn't have any builtin bounds checking.  If you want it, you've got to implement it yourself.  Take a look at those C code snippets again.  Together, the second and third function arguments determine how many bytes of data past the pointer you want to write.  You can put any non-negative numbers in there.  The compiler doesn't check whether it makes sense of not.  It's up to you.  Depending only your application, this can lead to some seriously buggy code.  



```zig 
fn transmuteSlice(comptime T: type, x: []T) []u8 {
 
    const num_bytes = @sizeOf(T) * x.len;
    const x0: [*]u8 = @ptrCast(x);
    const x1: [ ]u8 = x0[0..num_bytes];
 
    return x1;
}
``` 

```zig
fn transmutePtr(comptime T: type, x: *T) []u8 {
  
      const num_bytes = @sizeOf(T);
      const x0: [*]u8 = @ptrCast(x);
      const x1: [ ]u8 = x0[0..num_bytes];
  
      return x1;
}

``` 

```zig
const std    = @import("std");

const fs     = std.fs;
const print  = std.debug.print;
const mem    = std.mem;
const assert = std.debug.assert;


pub fn main() !void {
    
    const filename = "filename";

    var x0   = [_] f32 {1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0};
    const x1 : [ ] f32 = &x0;
    const x2 : [*] f32 = x1.ptr;
    const x3 : [*] u8  = @ptrCast(x2);
    const x4 : [ ] u8  = x3[0..(x0.len*@sizeOf(f32))];

    // Write the file
    try fs.cwd().writeFile(.{.sub_path = filename, .data = x4});

    
    // Read the file
    var b0 : [256] u8 = undefined;
    
    const y4 : [ ] u8  = try fs.cwd().readFile(filename, &buffer);
    const y3 : [*] u8  = y4.ptr;
    const y2 : [*] f32 = @alignCast(@ptrCast(y3));
    const y1 : [ ] f32 = y2[0..x0.len];

    // Compare the data written and the data read 
    assert(mem.eql(f32, x1, y1));

}
```

## Conclusion


