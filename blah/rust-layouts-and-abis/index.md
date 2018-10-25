% Notes on Type Layouts and ABIs in Rust

<span class="author">Alexis Beingessner</span>

<span class="date">October 9th, 2018 -- Rust Nightly 1.30.0</span>

Over the years I've found myself with a weird amount of knowledge about how types and ABIs in Rust work, and I wanted to write it all down in one place so that... it's written down in one place. Much of this information can or should be found in the [Rust Language Reference][reference] and the [Rustonomicon][nomicon].

Special thanks to Nicole Mazzuca for doing tons of fact-checking on this one!



# The Anatomy of a Platform

There are a lot of exotic platforms out there, and C is kinda jacked up from trying to support them all. Some of these distortions are annoying but technically fair, like not defining integers to be two's complement or not defining a byte (char) to be 8 bits, because those captured genuine differences between platforms at the time. Others are more just an artifact from C trying something that ended up being a mistake, like [the weird integer size fuzziness and promotion stuff][int-history].

A lot of the things C was trying to cope with have largely died off or been relegated to incredibly niche platforms. As such Rust took the opportunity to define more of the properties of the platforms it supports without breaking compatibility with C on those platforms.

> NOTE: this is not a normative document and the Rust devs haven't been very diligent in committing to these claims, so be a *bit* wary of relying on a property here that lacks a citation.

For Rust to [support a platform][rust-platforms] *at all*, its standard C dialect must:

* Have 8-bit, unaligned bytes (chars)
* Have a boolean be a byte, where `true = 1` and `false = 0`
* Have integers be [two's complement][twos-complement]
* Have [IEEE 754(-2008?) binary floats][ieee-floats], if they exist (e.g. we're comfortable with just disabling floats)
* Be at least 16-bit (just in terms of pointer size, I think?)
* Have NULL be 0 (although things may be mapped to 0, but that's messy since references can't be NULL)

(Additional constraints exist for running the actual standard library, like atomics support)

To a modern programmer, these are all incredibly reasonable constraints. In fact I expect most programmers would be very surprised if any of these things weren't true! To my knowledge the last great bastion of these properties being violated is some DSPs (Digital Signal Processors), because they really don't like 8-bit bytes. Rust is fine with not supporting those DSPs for the sake of making things cleaner for 99.9999% of its users.

Rust explicitly supports the following platform features, even though they're close to extinction:

* [Big-endian integers/floats][endianess]
* 16-bit pointers (although it appears that this is currently only really maintained by community volunteers to minimally support MSP430 microcontrollers)

And the following are *maybe* possible for Rust to support, but haven't really been sufficiently thought about, and it's likely we've made a decision that happens to mess these up (or should):

* [Segmented architectures][segmented]
* Platforms where `ptrdiff_t` = `intptr_t` = `ssize_t` doesn't hold





# The Anatomy of a Type

Types have several properties that define how they can be manipulated and accessed. It's possible to only know some of these properties, in which case it's only safe to do certain operations.

It's also possible to know literally nothing about a type, in which case the only thing you can really do with it is pass around pointers to it in a type-safe way. A situation where this might be true is when using a library which defines the type, and needs you to hold onto some pointers for it, but doesn't want you to actually access the data in those pointers. For instance this might be how state is passed to callbacks. Rust calls such a type an *[extern type][]*.

As of this writing, extern types are still experimental. `struct MyType { }` can be used for a similar purpose, although the compiler won't produce an error if you try to load/store values of that type, instead silently discarding the accesses.




## Size

The most fundamental property of a type is its *size*: how many bytes it occupies in memory. Knowing only the size of a type, it's possible to perform pointer offsets into arrays of that type and to copy values between pointers of that type. The stride of elements in an array is always equal to their size. Values of that type can also be loaded from or stored in registers, though registers generally aren't part of the semantic model of Rust.

In Rust, unlike C/C++, types may have a size of 0 (a zero-sized type, or ZST). This generally just means that it doesn't actually exist in memory, and therefore reads/writes of its values are no-ops.

A type's size may be a dynamic property of its values, as is the case for types like `[T]` and `Trait`. Such types don't implement the assumed-to-be-implemented `Sized` trait. Generic functions which wish to work with such a type must opt-in with `<T: ?Sized>`.





## Alignment

The second most fundamental property of a type is its *alignment*: what number of bytes its position in memory must be a multiple of (when stored in memory). So for instance a type with alignment 4 can only be stored at address 0, 4, 8, etc. With size and alignment, it becomes possible to allocate memory where values of that type can be stored.

Alignment is at least 1 and must always be a power of 2. Size is always a multiple of alignment. A type usually has the maximum alignment of its fields' alignments. Alignment requirements give rise to *padding* which is parts of the a type which are logically uninitialized because the size or relative position of something needed to be rounded to satisfy alignment. Reads to padding aren't guaranteed to produce reliable results, and writes to padding aren't guaranteed to be respected.

Alignment is largely an artifact of hardware which either prefers or requires that operations have a certain alignment. In a lot of cases misaligned accesses are "just" a nasty performance cliff, but in other cases the hardware will actually raise an exception for misalignment. In some sense how the hardware behaves doesn't actually matter anymore, because the compiler may assume pointers are aligned and miscompile your code if they aren't!

Zero-sized types may have an alignment greater than 1 (e.g. `[u32; 0]` has the alignment of `u32`, which is usually 4). Although ZSTs don't exist in memory, fields and pointers of that type must still be well-aligned, so a ZST may influence the layout, size, and alignment of a composite type that contains it.

As a slight aside, some older ABIs like the i386 System V ABI (the x86 linux C ABI) will align things in a slightly weird way. When placed in a struct, a `double` will be aligned to `4`, but on the stack it will always be aligned to `8`. However Rust is able to be compatible with this by just always aligning to `4`, as C can't tell if a pointer to a double is part of a struct or its own local.





## Offsets

The *offsets* of a type are the relative positions of each of its fields. There are three possibilities for offsets in Rust:

* Offsets are non-deterministic
* The *order* of offsets are deterministic, but their precise values aren't
* The exact values of the offsets are deterministic

Here the definition of deterministic is subtle. What I mean is that you could look at the struct and the target platform's definition and determine the offsets. By default, a user-defined Rust type's offsets are non-deterministic in the sense that different version of the compiler may choose different offsets, or subsequent builds may produce different offsets (though we will never link together two pieces of rust code that don't agree on the offsets of a type).

Here are a couple notable examples:

```rust
// The Rust compiler is not required to give these two structs
// the same offsets for their fields, even though they are identical.
struct A(u32, u64);
struct B(u32, u64);

// The Rust compiler is not required to emit the fields for this
// struct in the given order. e.g. it may put y before x in memory.
struct Rect {
    x: f32,
    y: f32,
    w: f32,
    h: f32,
}
```

There are two motivations for this: optimization and fuzzing.

In terms of optimizations, usually exact struct layout isn't something that is actually being relied on, so this is a fertile ground for easy optimizations. Especially for generic structs, where a single optimal layout for all type substitutions may not exist. For instance, this struct cannot have a single optimal ordering of its fields:

```rust
struct Impossible<T, U, V> {
    t: T,
    u: U,
    v: V,
}
```

Consider substituting `u16`, `u16`, and `u32` for T, U, and V. The struct will be tightly packed as long as the `u32` is not the second element. However any ordering we choose must make *some* element be in the middle, and then we may change that type to u32 to make the ordering suboptimal. Therefore there is no optimal single ordering for fields in generic structs.

The fuzzing motivation (which to date has not been taken advantage of) is to allow field orderings to be randomized to more readily expose latent bugs.

As will be discussed in later sections, certain annotations will induce a deterministic field ordering. But if a field has a type which doesn't have a deterministic ordering, its size may also be non-determinstic, and that may lead to the outer type still having non-deterministic offsets.

So for instance, this struct has a deterministic ordering of its fields, but not deterministic offset values:

```rust
#[repr(C)]
struct MyStruct {
    x: u32,
    y: Vec<u8>,
    z: u32,
}
```

`Vec` doesn't have any deterministic ordering, so although we deterministically know the exact offsets `x` and `y` will be stored at, we can't know the offset of `z` or the size of `MyStruct`, because those facts depend on the size of `y` which isn't deterministically knowable. As such this type isn't suitable for FFI with C.

Ok actually it might be the case that alignment also isn't deterministic by default? In which case `y`'s offset would also be unknown. This is under-defined, and [is actively being discussed by the Rust developers][repr-rust].



## Layout

The *layout* of a type is its size, alignment, offsets, and the recursive layouts of its fields.

Having the full layout of a type allows one to access the fields of a type. It also makes it possible to convert between types with *compatible* layouts. There isn't really a strict definition of compatible layout I can come up with. Basically if you know some memory has the same location in both types, you can reinterpret one type as the other and know what happens to that memory. This is perfectly legal in Rust because we have no type-based alias analysis (TBAA, AKA "strict aliasing").

For instance, you can create your own kind of inheritance this way:

```rust
#[repr(C)]
struct Base {
    x: u32,
    y: u64,
    is_derived: bool,
}

#[repr(C)]
struct Derived {
    base: Base,
    z: f32,
}

fn process<'a>(data: &'a Base) {
    print!("x: {}, y: {}", base.x, base.y);

    if data.is_derived {
        // upcast from Base to Derived
        let derived = unsafe { mem::transmute::<&'a Base, &'a Derived>(data) };
        print!(", z: {}", derived.z);
    }

    println!("");
}
```

Also if you can create a type declaration with compatible layout in C/C++, you can pass that value across the FFI boundary by-reference and have both sides be able to read/write all the fields.



## ABI

The layout of a type is enough to do anything you want to do with a type *within Rust*, but it's insufficient for full communication with C. In particular, it's insufficient for passing things *by value* to a C function. This is because there are additional properties that define the *ABI* (Application Binary Interface) of a type. The ABI of a type determines how it is passed to a C function by-value ([see the section on calling conventions for details][calling-conventions]).

To my knowledge the only property that is unique to ABI is that of *type-kind*. Although `#[repr(C)] struct MyType(u32)`, `u32`, and `f32` may be layout compatible on a given target, they may still have incompatible ABIs because they have a different type-kind.

As of this writing, there are 4 type-kinds that Rust can care about:

* Integer (pointers are treated as integers here, though this may change in the future)
* Float
* Aggregate
* Vector

> NOTE: type-kind is a non-normative concept that makes talking about ABIs clearer to me. All of this could be correctly specified without appealing to it. Although it *is* similar to the concept of a type's "class" used in [sysv x64 ABI section 3.2.3][sysvx64-abi].

The integer and float type-kinds represent the two kinds a primitive can have. If two types have the the same size, alignment, and primitive type-kind, then they are completely ABI compatible (e.g. u64 and usize have identical ABIs on x64 linux).

The aggregate type-kind is the default for any struct, enum, or union. However aggregate type-kind can be changed to any of the other 3 under the right conditions and with the right annotations. This will be detailed in a later section.

All C structs and unions have the aggregate type-kind. C SIMD types have the vector type-kind. C enums have integer type-kind.

The precise ABI of aggregates and vectors depends on the precise ABIs of their fields. So for instance, I believe these two types have different ABIs on x64 linux even though they have identical size, alignment, and type-kind:

```rust
#[repr(C)]
struct Homo(u64, u64);

#[repr(C)]
struct Hetero(u64, f64);
```





## The Layouts/ABIs of Builtins

Here is a table of the ABIs of the core primitives in Rust, which C/C++ types they are guaranteed to be ABI compatible with, and what values are defined for these types (storing other values in such a type may lead to Undefined Behaviour):

|                | size | align | kind    | C/C++ type        | defined values              |
|----------------|------|-------|---------|-------------------|-----------------------------|
| u8             | 1    | 1     | integer | uint8_t           | all                         |
| u16            | 2    | ≤2    | integer | uint16_t          | all                         |
| u32            | 4    | ≤4    | integer | uint32_t          | all                         |
| u64            | 8    | ≤8    | integer | uint64_t          | all                         |
| u128           | 16   | ≤16   | N/A     | N/A               | all                         |
| usize          | ptr  | ptr   | integer | uintptr_t         | all                         |
| i8             | 1    | 1     | integer | int8_t            | all                         |
| i16            | 2    | ≤2    | integer | int16_t           | all                         |
| i32            | 4    | ≤4    | integer | int32_t           | all                         |
| i64            | 8    | ≤8    | integer | int64_t           | all                         |
| i128           | 16   | ≤16   | N/A     | N/A               | all                         |
| isize          | ptr  | ptr   | integer | intptr_t          | all                         |
| *const T       | ptr  | ptr   | integer | T*                | all                         |
| *mut T         | ptr  | ptr   | integer | T*                | all                         |
| &T             | ptr  | ptr   | integer | T*                | not null                    |
| &mut T         | ptr  | ptr   | integer | T*                | not null                    |
| Option<&T>     | ptr  | ptr   | integer | T*                | all                         |
| Option<&mut T> | ptr  | ptr   | integer | T*                | all                         |
| bool           | 1    | 1     | integer | bool (_Bool)      | 0=false, 1=true             |
| char           | 4    | ≤4    | N/A     | N/A               | 0x0-0xD7FF, 0xE000-0x10FFFF |
| f32            | 4    | ≤4    | float   | float             | all                         |
| f64            | 8    | ≤8    | float   | double            | all                         |

In theory `u128` and `i128` should match the ABI of `__int128` but [they don't right now due to a bug in llvm][llvm-int128]. Similarly we *could* probably define Rust's `char` to match C++'s `char32_t` but so far no one has cared enough to look into the details and champion the issue.

Note that in practice primitives are *usually* aligned to their size. A smaller alignment is often an indication that the type is software-emulated on that platform (e.g. `u64` has align 4 on x86 linux). Ultimately the size and alignment is just "whatever the target's standard C implementation does", as compatibility is our primary concern here.

Arrays (`[T; n]`) have the same layout as C arrays: aligned to `T`, with `n * size_of::<T>()` size, and element `i` is at byte offset `i * size_of::<T>()`. However arrays currently have no specified type-kind, as arrays cannot actually be passed by-value in C (`void func(int x[5])` is semantically identical to `void func(int* x)`).

Tuples have completely unspecified layout, except for `()` which is size 0 and align 1.

I think that covers all the interesting builtins.






## Specifying Layouts and ABIs

The following annotations have the following effects on layout and ABI:

* `#[repr(C)]` on a struct forces the fields to be laid out in the order of their declaration with the same greedy padding rules as C. If all the fields have a fully defined layout, then the type has a fully defined layout. Note that this annotation has pure-rust applications such as type-punning or inheritance, and that `#[repr(C)]` doesn't actually guarantee that the type is FFI-safe.

* `#[repr(simd)]` on a struct is the same as `#[repr(C)]`, except it gives the type the vector type-kind. This feature is currently unstable without any proposed path to stabilization. Longterm, stable vector-kind types should be available in the stdlib. Short-term they are unstably available in the `packed_simd` crate.

* `#[repr(transparent)]` on a struct with a single field gives it the ABI of its field. So if it contains an `i32`, it has the ABI of `i32`. Generally this is only necessary to get a matching type-kind, as size and alignment will otherwise naturally match the field. This is especially useful for making FFI-safe newtyped integers (like for typed units).

* `#[repr(packed(N))]` on a struct removes any non-trailing padding and sets the type's alignment to N, making it compatible with packed structs in C. Note that this will lead to fields being misaligned. Direct accesses to the fields will generate code to manage this misalignment, but taking pointers to these fields is dangerous as the compiler will "forget" that they are misaligned and subsequently assume that they are, leading to Undefined Behaviour.

* `#[repr(align=X)]` on a struct forces the struct to be aligned to at least `X`. This may in turn affect size.

* `#[repr(C)]` on a fieldless enum makes the enum have the same ABI as a C enum with the same declaration. So it will have integer type-kind and the size and alignment of whatever integer type the target's C enums are desugarred to (usually `int`?). Note that unlike C, all unspecified values are still forbidden.

* `#[repr(int)]` (where `int` is any of the primitive integer types, such as `u8`) on a fieldless enum makes the enum have the same ABI as the given integer type. This is useful for matching the ABI of `enum MyEnum: some-int-type` in C++. Note that unlike C, all unspecified values are still forbidden.

* `#[repr(int)]` or `#[repr(C)]` on an enum with fields will give it [a defined C-compatible tagged union representation][really-tagged-unions].



And that's everything I know about defining type layouts and ABIs in Rust!



# Extended Random Notes

Welcome to The Turbo Footnotes, where I just dump random stuff that is tangentially related.





## The C Integer Hierarchy

So C had two problems it was trying to solve: different platforms have different values for the size of a byte (the smallest unit of addressable memory), and different platforms have different "native" (most efficient/important) integer sizes.

Their solution to this was two-fold: define a type for the platform's unit of memory (`char`) and then define a hierarchy of integers with different sizing constraints between them. In this way code could theoretically be portable and run reasonably well on 10-bit-byte platforms, 16-bit platforms, 32-bit platforms, and so on.

The constraints for the core integer types are as follows:

* `char` is at least 8 bits, and all other types must be a multiple of its size (`CHAR_BIT`)
* `short` is at least 16 bits, but also at least a `char`
* `int` is at least a `short` (intended to be the "native" integer size)
* `long` is at least 32 bits, but also at least an `int`
* `long long` is at least 64 bits, but also at least a `long`

So on the surface this is a fairly reasonable hierarchy: if you want 16-bit value, use a short. If you want a 32-bit value, use a long. They might be bigger, but that's probably fine... right?

Well no because it turns out exact size is kinda important! For instance if you need to read/write exactly 32 bits out of some binary format, how are you supposed to do that? If you use a `long`, it could access 64 bits! Also which of these is suitable to store the bits of a pointer? (`intptr_t` was only added in C99!)

This isn't just a theoretical concern. During the 32-bit era, assuming `int` was *exactly* 32 bits became so rampant that when 64-bit hardware started to show up, compiler developers were forced to define `int` to still be 32 bits, as too much software was completely busted when `int` was anything else.

Of course the whole point of `int` is that its *supposed* to be the native integer size, so this in turn pushed compiler developers to abuse the fact that signed overflow is undefined to allow `int` to be implicitly promoted to 64-bit in places where it really mattered.

There was a really great compiler dev (gcc?) email about this history but I can't seem to find it anymore. So for the time-being I'll settle for [Fabien Giesen's discussion on the matter][ryg-ints].





## Endianness

For integers and floats, endianness (AKA byte-order) specifies how the individual bytes of the value are ordered. In a big-endian encoding they're written out like you would write numbers on paper: the most significant bytes come first. In a little-endian system the least significant bytes come first. As far as I can tell this is basically the systems programming version of oxford commas, in that it really doesn't matter much but everyone has strong opinions so you regularly see both.

These days little-endian has largely won the battle for what new hardware uses as its native format (e.g. all x64 chips and most ARM chips), while big-endian has been relegated to being the storage/wire encoding for a bunch of random formats (e.g. IPv4, IPv6, TCP, and UDP, etc.).

With that said, it's really easy to write programs that are agnostic to the native endianness of a platform, so it's really not a big deal for Rust to support the remaining big-endian platforms (note: `unsafe` Rust features like `transmute`, `union` type punning, etc. can introduce endian-dependent behavior). 





## Segmented Architectures

A segmented architecture, for our purposes, is one in which pointers that have an identical runtime representation may actually refer to different regions of memory, because they are associated with different *segments*.

One example generously provided to me by John Regehr is the [ATmega128][], an 8-bit arduino microcontroller which has 4 segments: SRAM, EEPROM, ROM, and I/O.

I am aware of three reasons why segmentation can be messy to the programming model:

* Pointers to segments may have different properties/requirements.
* How pointer equality/comparison between pointers to different segments should work is unclear.
* Segmentation may involve decoupling the size of a pointer and the size of a pointer offset, which Rust currently mandates are equal (usize).

Unfortunately I am running out of steam here and really am only peripherally aware of these issues, so uh, I'm just gonna leave it at that for now. Someone else figure this out!




## Calling Conventions

ABI can mean a lot of different things to different people. At the end of the day it's a catch-all term
for "implementation details that at least two things need to agree on for everything to work". In this document we refer to ABI as covering type layout and how the different types/values are passed between C functions, as these are the aspects of the Rust ABI that are guaranteed and useful.

There are additional details of Rust's ABI which are currently unspecified and unstable, such as vtable layouts for trait objects, and how linker/debug symbols are mangled. It's fine if that was gibberish to you, because you aren't really allowed to care about those things right now! (Although that doesn't necessarily stop people from trying...)

Anyway, here I want to zoom in on *calling convention*, which is the argument/return-passing aspect of ABI, since I received a fair amount of questions about it.

For the sake of simplicity I'm only going to focus on the things relevant to calling conventions in C on popular modernish hardware and OSes (read: x86, x64, and AArch64; Mac, Windows, Linux, Android, and iOS). Some exotica off the top of my head that I won't be covering, but might be interesting to you:

* [Lisp machines][lisp-machines]
* [Stack machines][stack-machines]
* [Segmented Stacks in Go][go-stacks]
* [Swift's ownership/ARC ABI][swift-arc]
* [Non-trivial types in C++][cpp-trivial]



### Problem and Motivation for Calling Conventions

So first off, the problem: it's pretty common for CPUs to have some kind of native notion of calling a function, but it's generally much simpler than a programming language function call. In its simplest form, a call instruction just tells the CPU to jump to a new set of instructions and start executing those. But functions as we know them in most languages have arguments, and so we need to define some way for the *caller* to set up state so that the *callee* can find those arguments. Function returns are similar, requiring the *callee* to set up state so that the *caller* can pick up where it left off, and also acquire any returned values.

There's two major ways to pass state between the two sides of a function call: in registers, and on the stack. There's a bunch of competing concerns here that make one of the other more desirable and I don't pretend to fully understand them, but I'll try to give a sketch of some of them here.

[Registers][registers] are your CPU's primary observable global (well, thread-local-ish) state. They're incredibly fast to access, but also generally very small. Also registers are usually mandatory to get anything done. CPU instructions can be thought of as little builtin functions with their own adhoc ABIs, and those ABIs generally pass arguments/returns in registers. A nice beefy modern cpu might give you general purpose registers on the order of 32 64-bit values. Less than a KB of working space! SIMD registers might beef this up to a few KB, but they're also much less flexible to use. See also [register renaming][register-renaming] for fun details on how the size of your working set is a lot more complicated than just a number!

Values are passed in registers by just... having them be there! If an argument should be passed in register 1, the caller ensures that value is in register 1 before performing the call, and when the callee starts up it knows register 1 holds that value. Similarly if something should be returned in register 1, the caller just ensures that value is in register 1 before returning control to the callee.

[The stack][stack] is a simple abstraction for extending the working set for your thread with RAM. Stacks are contiguously allocated with some fixed maximum size that should be much larger than registers (often on the order of 8MB these days). In its simplest form, when a function is called it requests that the stack "push" enough space to fit all the state it might need upfront, and when it returns that size is "popped" off. This chunk of space each function requests is known as a *stack frame*. For fun complications, see [alloca][alloca] and [the red zone][red-zone].

The precise details of pushing and popping frames is another thing specific to the calling convention, but I don't think we need to concern ourselves with those details for this discussion. All we need to know is that the stack is predictable enough that values can be passed between functions on the stack by either putting values at the end of the callers stack frame or at the start of the callees frame. In either case the function that isn't responsible for storing the values may assume that enough space for the arguments or return value is in the other's frame, and freely read and write that memory as required.

The main tradeoff for the stack's size is that using it is usually going to be slower than registers. Although as always with modern hardware, that's a complicated matter due to the magic of [caches][caches] and [speculation][speculation]. Regardless, let's proceed under the assumption that keeping stuff off the stack and in registers is ideal.

Note also that, to avoid copying large values into the right registers or right place on the stack, we may instead simply pass a *pointer* to that value (either on the stack or in a register), even if the function signature otherwise suggests that it should be passed by-value. This is particularly effective in cases where a large value gets used in a series of function calls.

The last thing we need to keep in mind to understand calling conventions is our constraints. We need our ABI to work in a fully *virtual* or *dynamic* context. That is, the only thing that the caller and callee both know is the signature of the callee. It must be possible for any other function to call the callee, and for the callee's implementation to be swapped out between calls (such as with vtables or [dynamic linking][dynamic-linking]).

Before we even get into the issue of passing arguments, everything we now know leads to a conflict: both functions want to use registers as much as possible to Go Fast, both functions have the same registers, and neither has any idea which registers the other is actually using!

Here's one very simple (bad) solution: at the start of the callee, save *all* the registers to the stack. Then when the callee is ready to return, restore all the registers from the stack. We call this idea of the callee preserving registers *callee saving* or *non-volatile registers*. This solution is pretty bad because, well, the registers are pretty big! Copying all that data to and from the stack takes a bunch of time. We can do better. (Total aside: this is how [context-switching][context-switching] works, although OSes have tricks to avoid saving/restoring all the registers all the time.)

Here's a slightly better solution: right before performing the call, the caller saves all the registers it actually cares about to the stack. Then the callee can assume it can do whatever it wants with the registers, and the caller just assumes the callee has stomped over all the registers, and reinitializes them as it sees fit. This is *caller saving* or *volatile registers*. By default, this is a lot better because the caller generally won't actually be using many of the registers (especially because most of the register size is in harder-to-use SIMD). Another advantage of this approach is that it now completely frees up all the registers to be used for argument/return value passing!

Modern calling conventions generally have a more hybrid approach which, presumably, is based on typical patterns. Some registers are marked as callee-saved, while others are caller-saved. This gives the callee and caller some flexibility to try to cooperate to avoid register saving.

For instance, if the caller keeps all of its working set in non-volatile registers while the callee keeps all of its working set in volatile registers, then no registers need to be saved at all. This gives a reasonable motivation to keep around a *few* callee-saved registers. Similarly, callee-saving is desirable for code that passes around a "context" pointer to lots of functions (see: `this`/`self` in a huge amount of languages, and I assume several Very Cursed C Frameworks).




### Some Examples of Calling Conventions

Rather than detail entire calling conventions here, I'm mostly just going to focus on where the distinctions made in the earlier sections of this document affect how different conventions behave. In particular I believe it is sufficient to look at some examples of value passing for the System V ABIs for [x86][sysvx86-abi] ("cdecl") and [x64][sysvx64-abi] (these are the standard Linux/BSD/MacOS calling conventions, although x86 was a bit more wild-west so assume I'm talking about GCC on Linux here while I pray that that has any kind of consistent meaning).

A note on notation: I use `stack -x` to indicate that the value is stored `x` bytes before the frame of the callee (because the System V ABIs store stack arguments in the caller).

Given this decl:

```C
struct Meter { int32_t len; }
struct Point { int32_t x; int32_t y; };

int32_t process(void* a, float b, struct Meter c, struct Point d);
```

We get these lowerings:

```text
process (x86 System V):
a           void*:  stack -4
b           float:  stack -8
c         {int32}:  stack -12
d  {int32, int32}:  stack -20
----------------------------------
return      int32:  register eax
```

```text
process (x64 System V):
a           void*:  register rdi
b           float:  register xmm0
c         {int32}:  register rsi
d  {int32, int32}:  register rdx
----------------------------------
return      int32:  register rax
```

Right away we see the older ABI mostly just passes things on the stack, while the newer ABI aggressively passes things in registers. Note that the `float b` argument is passed in the `xmm` registers instead of the general purpose `r` ones, as floats and integers are treated differently (motivating our distinction of the two).

The distinction between a composite and primitive is motivated by how return values are handled in the x86 ABI. If we change `process` to return a `Meter`, we get the following:

```text
process (x86 System V):
return    {int32}:  stack -4
a           void*:  stack -8
b           float:  stack -12
c         {int32}:  stack -16
d  {int32, int32}:  stack -24
----------------------------------
return   {int32}*:  register eax (pointer to stack -4)
```

```text
process (x64 System V)
a           void*:  register rdi
b           float:  register xmm0
c         {int32}:  register rsi
d  {int32, int32}:  register rdx
----------------------------------
return    {int32}:  register rax
```

Even though the layout of the type is identical, the x86 ABI always passes structs and unions on the stack as an implicit first argument. The x64 ABI "fixes" this, and just treats the two identically.

However the x64 ABI is very complex in how it passes composites by-value. Consider these two declarations:

```C
struct Ints         { int32_t a; int32_t b; int32_t c; int32_t d; };
struct IntAndFloats { int32_t a;   float b;   float c;   float d; };

void process1(struct Ints vals);
void process2(struct IntAndFloats vals):
```

```text
process1 (x64 System V)
(vals.a, vals.b):  register rdi
(vals.c, vals.d):  register rsi
```

```text
process2 (x64 System V)
(vals.a, vals.b):  register rdi
(vals.c, vals.d):  register xmm0
```

The x64 ABI chunks structs into 8-byte chunks and does a recursive classification of the fields. In this case, we see that for the first half of `IntAndFloats` the integer `a` "dominates" the float `b`, and so that chunk is passed in general purpose registers. However the second chunk consists entirely of floats, and so is passed in `xmm0`. This shows us that we need to know the exact ABIs of all of the fields of a composite to properly pass it in the x64 ABI.




[reference]: https://doc.rust-lang.org/reference/
[nomicon]: https://doc.rust-lang.org/nomicon/
[ATmega128]: http://www.kjit.bme.hu/images/stories/targyak/jarmufedelzeti_rendszerek/atmel_atmega128_manual.pdf
[extern type]: https://github.com/rust-lang/rfcs/blob/master/text/1861-extern-types.md
[really-tagged-unions]: https://github.com/rust-lang/rfcs/blob/master/text/2195-really-tagged-unions.md
[ryg-ints]: https://gist.github.com/rygorous/e0f055bfb74e3d5f0af20690759de5a7
[int-history]: #the-c-integer-hierarchy
[llvm-int128]: https://github.com/rust-lang/rust/issues/54341
[sysvx86-abi]: https://www.uclibc.org/docs/psABI-i386.pdf#section.2.2
[sysvx64-abi]: https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf#section.3.2
[rust-platforms]: https://forge.rust-lang.org/platform-support.html
[twos-complement]: https://en.wikipedia.org/wiki/Two%27s_complement
[ieee-floats]: https://en.wikipedia.org/wiki/IEEE_754
[endianess]: #endianness
[segmented]: #segmented-architectures
[repr-rust]: https://github.com/rust-rfcs/unsafe-code-guidelines/issues/11
[register-renaming]: https://en.wikipedia.org/wiki/Register_renaming
[stack-machines]: https://en.wikipedia.org/wiki/Stack_machine
[lisp-machines]: https://en.wikipedia.org/wiki/Lisp_machine
[go-stacks]: https://blog.cloudflare.com/how-stacks-are-handled-in-go/
[swift-arc]: https://github.com/apple/swift/blob/edfb86f09aaf8cfcc1a0608bdcd6fe21b7a35460/docs/CallingConvention.rst#responsibility
[alloca]: http://man7.org/linux/man-pages/man3/alloca.3.html
[red-zone]: https://en.wikipedia.org/wiki/Red_zone_(computing)
[registers]: https://en.wikipedia.org/wiki/Processor_register
[stack]: https://en.wikipedia.org/wiki/Call_stack
[caches]: https://en.wikipedia.org/wiki/CPU_cache
[speculation]: https://en.wikipedia.org/wiki/Speculative_execution
[dynamic-linking]: https://en.wikipedia.org/wiki/Dynamic_linker
[context-switching]: https://en.wikipedia.org/wiki/Context_switch
[cpp-trivial]: https://quuxplusone.github.io/blog/2018/05/02/trivial-abi-101/
[calling-conventions]: #calling-conventions
