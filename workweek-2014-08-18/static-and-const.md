# 2014-08-20 Static and const

https://github.com/nikomatsakis/rfcs/blob/const-vs-static/active/0000-const-vs-static.md

- nmatsakis: Sent out an RFC a while back with a proposal to change our approach to static data.
- nmatsakis: The goal is (1) to be a little more honest about the categories of static things (2) to enable use patterns of constants that don't work today (e.g. initializer for an atomic int as a constant -- hard to do) (3) to make the rules a little more principled.
- nmatsakis: The proposal is: add const as a concept. It represents a value, not an address -- it's not a variable. If you reference it later, it's an rvalue that's instantiated at that point. Quite different from what a static does. But quite similar to nullary enum variants/structs.
- nmatsakis: NIce thing is, you don't have to restrict the types. We were worried about interior mutability.
- nmatsakis: To prevent that, we had some rules like: you can't take the address of a static if it has interior mutability.

```
const name: Type = initializer;

...

let x = name;
let y = &name; // allocate space on my stack and copy name into it
```

- nmatsakis: Essentially no limitations on const types. const covers almost all use cases of wanting global data
- nmatsakis: statics, then, are really about declaring global variables -- statically allocated memory. If you take the address, you get back a reference with 'static lifetime.
- nmatsakis: A static variable must be Sync, so that you can't create data races with a cell
- nmatsakis: All access to static mut is unsafe, and there are no restrictions
- nmatsakis: One additional detail: if you take the address of a constant in a static initializer -- there's no stack to put it on. The rule would be: you can only do this if the const has no interior mutability, and it's as if you introduced a static temporary:

```
struct SomeStruct { x: uint }
const Foo: &'static SomeStruct = &SomeStruct { x: 1 };
// equivalent to:
static _tmp: SomeStruct = SomeStruct { x: 1 };
const Foo: &'static SomeStruct = &_tmp;
// requires that the type does not contain interior mutability (UnsafeCell)
```

- nmatsakis: Motivation is to build up constants with indirections. Does what you expect -- allocate some static space. read-only memory in the binary
- nmatsakis: It's also invisible how many statics are created, so you could coalesce.
- nmatsakis: The only use cases of statics: global locks or atomic counters. You probably only want static mut for FFI reasons, or perhaps testing
- acrichto: Huge breaking change, but basically all statics today would become consts, pretty mechanical
- nmatsakis: The compiler might need to grow ability to grow stack space for constants
- acrichto: I think we can handle this as-is, maybe not efficiently
- pcwalton: Would const &'static str work?
- nmatsakis: Yes. Almost always use const
- pcwalton: My only concern is that people would have to think too hard, sounds like they won't; +1
- pcwalton: I think it's important for the embedded use-case to have statically-allocated ref cells. Missiles don't allocate, by and large, because allocation can fail and missiles can't. A lot of people write all their code with static allocations everywhere.
- nmatsakis: That's a use case for static mut
- zwarich: Is Rust really feasible for people to write programs with no allocation
- pcwalton: No reason to make it hard. You'd have to ref cells, I guess
- nmatsakis: You could use static mut
- acrichto: After you get the pointer, don't need unsafe
- nmatsakis: You could write a type that declares itself Sync, but doesn't sync
- nmatsakis: I'll revise the RFC for &'static and post publicly






















