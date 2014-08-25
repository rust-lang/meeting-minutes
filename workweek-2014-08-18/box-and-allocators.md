# 2014-08-21 Box 

- nmatsakis: I want to clarify plans for the box keyword; hopefully can result in an RFC
- nmatsakis: The first part is the allocation syntax below:

```
box expr2 OR:
box(expr1) expr2
where expr1 defaults to Heap if not otherwise specified
```

- nmatsakis: You write box and an expression, and you can optionally use parens to specify "the boxer"
- nmatsakis: The above then desugars using the following trait:

```
trait Boxer<INPUT> {
    type OUTPUT;
    
    fn make_box<T:FnOnce() -> INPUT>(self, input: T) -> OUTPUT;
    //            ^~~~~~~

    fn make_box<T:|:| -> INPUT>(self, input: T) -> OUTPUT;
}

box(expr1) expr2
// desugars to:
Box::make_box(expr1, || expr2)
  // or |:| expr2 if we have that syntax

box(Rc) 1 -> Rc<int>
box(Heap) 1 -> Box<int>
box 1 -> Box<int>

struct RcBoxer;
const Rc: RcBoxer = RcBoxer;
impl<T> Boxer<T> for RcBoxer {
    type OUTPUT = Rc<T>;
    
    fn make_box(self, v: FnOnce() -> T) {
        let tmp = /* allocate */;
        *tmp = v();
        return tmp;
    }
}
```

- nmatsakis: You have to make a named struct, just so you have a type to hang the impl off
- nmatsakis: The main purpose of the syntax is the "once" closure above, which allows you to build the box in place. The closure defers the actual execution.
- nmatsakis: The make_box routine has to clean up `tmp` if there's a failure, etc. Currently the compiler does this, but it'd move into library code.
- nmatsakis: The other part is box patterns:

```
match foo {
    box pat => { ... }
}

// for a box pattern, type of pattern being matched must implement
// Deref, and `pat` will be matched against the deref result
```

- nmatsakis: One implication is that you always write `box` for the pattern and it works for all different kinds of smart pointers.
- nmatsakis: There's also an interaction with allocators (but that's more of a convention)
- pnkfelix: Why use the Deref trait, rather than unboxer?
- nmatsakis: That might make sense; just wanted to avoid a proliferation of random traits. Maybe the boxer trait would have a reverse operation
- aturon: This feels like `unapply` in Scala; we could go down that road
- nmatsakis: If we did general purpose unapply, I wouldn't want to tie it to `box` keyword
- aturon: Yes, definitely
- stuart: The intent is to be able to move out of the box?
- nmatsakis: Actually, you'll want multiple traits representing different ownership semantics
- luqman: How does it work with Deref? You'd have to write box ref x?
- nmatsakis: Yes, as today. We'll have to work on inference.
- nmatsakis: In the typechecker, we use normal deref to get the type, and then come back and determine the ownership variant to use.
- nmatsakis: So that's one reason to tie to deref, but I don't feel strongly
- nmatsakis: Allocators fit in by a convention on how the expressions are used:

```
struct RcIn<A:Alloc>(A);

impl<A:Alloc> Boxer for RcIn<A> { ... }

box(RcIn(alloc)) foo
```

- nmatsakis: I'm a little torn about `box`; maybe we could just do something with deref patterns alone. The downside is that the expression that you want to put into the box gets evaluated first, put on the stack, and then put into the box. LLVM can't optimize this. Maybe we could eventually work around that.
- aturon: And the downside of `box` is just more "stuff"?
- nmatsakis: Yeah. Probably not a big deal. It's a little quirky, too: the parens are significant
[...some discussion of unboxed closure syntax...]
- nrc: what about arena allocation?
- nmatsakis: Just a custom allocator, I think. An arena would be an allocator. Maybe it can also implement boxer for convenience.
- nrc: There's nothing in the traits that ties you to using a struct? You could return a reference?
- nmatsakis: Yes, I believe so. There's no connection between the input/output types in the traits (would be different with HKT)
- nmatsakis: In this design, you can have a boxer that doesn't work for all types T
- nrc: you'd write `box (TheArena) e`?

Random syntax thought

```
box(in TheArena) expr //-> &T
box(in Rc) expr
```

- steveklabnik: It seems like arenas and things like Rc are different. The Rc tells you about a single reference, but an arena covers multiples.
- nmatsakis: The fact that you can use an arena as a boxer is not totally right. Really, it's an allocator (a pool of memory), and the smart pointer is just ownership patterns laid over some memory source. You could have an Rc over an arena -- doesn't make a lot of sense in that case, but maybe you care about the destructor or something.
- nmatsakis: But there is a distinction, and arenas can play both roles, which can be confusing. But a box allocated in an arena makes sense.
- aturon: to recap, benefits:
  - creating boxes allows you to create value in place
- aturon: pattern matching too but that could be obtained via some other mechanism
- aturon: But if we have creation mechanism, having destructuring along side it seems fine -- that's what we usually do
- steveklabnik: People don't seem worried about the syntax -- the confusion is more about implementing boxers
- nrc: What's the relation to 1.0? Does any of this block 1.0?
- nmatsakis: No.
- brson: What about the box syntax for Rc? right now people use Rc::new
- nmatsakis: We should extend the fixed set to include Rc, even if it desugars to Rc::new
- aturon: Are we going to drop Rc::new? Or is this just a style thing?
- nmatsakis: Just style
- brson: But style is important -- effect the "feel" of the language
- pcwalton: Scope creep!
- steveklabnik: Right now, it's hard to explain to people what they should use -- when to use box versus new...
[... some discussion of pre 1.0 implementation strategy...]
- pnkfelix: Why not do the proper desugaring now?
- nmatsakis: At that point it's just an overloaded operator (except for the implicit closure)
- nmatsakis: We can just do this as a front-end rewrite
- aturon: Including pattern matching?
- nmatsakis: No, this is all for construction only. box patterns would truly be scope creep. It'd be nice, but not essential, they come up much less often
- acrichto: The make_box function should be itself unboxed, right?

Conclusion: we can implement the construction form today. 
Punt on pattern form for now
nmatsakis to write RFC

- acrichto: What about freeing the allocation on exit?
- nmatsakis: We probably want a "finally" construct for this

# Allocators

- nmatsakis: The motivation is: people want to allocate their memory using different sources. Default is malloc, but you may want arenas, accounting, etc.
- nmatsakis: Following EASTL (and strcat's proposal), we'd have a trait like the following that describes raw memory allocation:

```
trait RawAlloc {
    unsafe fn raw_alloc(&self, size: uint, alignment: uint) -> *mut u8;
    unsafe fn raw_free(&self, size: uint, alignment: uint, ptr: *mut u8);
    unsafe fn raw_realloc(...);
}
```

- nmatsakis: Basically C API, but with alignment, and size on free
- nmatsakis: The tricky thing is: how to make this future-proof for a GC
- nmatsakis: The idea is that the raw allocation functions are `unsafe` and not intended for direct consumption. Instead, when you want to allocate, you should use the `alloc` top-level function, which takes an allocator and a type: (DST may need some variations)

```
unsafe fn alloc<A:Alloc,T>(alloc: &A) -> *mut T {
    if !type_contains_gc_roots::<T>() { alloc.alloc(...) }
    else {
        ...
    }
}
unsafe fn free<A:Alloc,T>(alloc: &A, ptr: *mut T);
```

- nmatsakis: The reason for this indirection is that these high-level versions know the type that you're allocating. Internally, they use an intrinsic to tell whether the type contains GC roots.
- nmatsakis: If there are no roots, just call the raw functions -- zero overhead.
- nmatsakis: Otherwise, they use pools etc to do whatever we need for tracing later on.
- acrichto: Does it suffice for A to only have the alloc trait (for GC)?
- nmatsakis: I think so. The plan is: you can copy Gc types all you want. When you want to collect, we scan the stack to find one set of roots, and then we scan all the allocated objects that might have managed data. (Note that you wouldn't use this `alloc` for the Gc pointers themselves). This design assumes a single Rust GC, rather than user-pluggable ones.
- nmatsakis: To work with DST, you need intrinsics to work with possibly fat pointers; you need that to write Rc that uses the alloc API and still know the amount of space (or maybe you just need this for the alloc function). Just stuff to query the size of a type given a fat pointer.
- acrichto: So RawAlloc would be taken as a type parameter to collections?
- nmatsakis: Yes.

- nmatsakis: A different part of the proposal is the behavior of the default raw allocator. We could just hard wire to jemalloc. But then, if I have my own allocator and want to use a library that's not generic over the allocator, they'll wind up with jemalloc rather than mine.
- nmatsakis: We could make it easier by instead having the default allocator to check TLS for the allocator to use for the thread. You'd have a virtual call on every allocation, but you could switch allocators on a per-thread basis, dynamically.
- nmatsakis: It's a flexibility/perf tradeoff
- nmatsakis: If you didn't want this overhead, you could at link time provide your own default, fixed allocator.
- nmatsakis: A common pattern in C libraries is to take malloc/free as function pointers, so there's precedent
- nmatsakis: The fact that you can pick this at link time means we could pick hard-wired as the default, and opt-in to this dynamic version.
- acrichto: This is for interoperating with Rust or C code?
- nmatsakis: Rust. If someone's library just asks for a new vector, they'll get the default allocator, and I can't override it.
- nmatsakis: Of course, someone else's code could hard code their own allocator, and you can't change that.
- nmatsakis: Finally, we could probably add this later, backwards-compatibly.
- pnkfelix: To be clear, you do have to stick with the same allocator throughout a thread's lifetime.
- acrichto: We already have problems about child tasks not inheriting settings from parents
- nmatsakis: Also migration across threads. Maybe this just doesn't work.
- aturon: if the motivation for dynamic allocator lookup is ergonomics, we could improve things to make the threading of a generic allocator more ergonomic.
- nmatsakis: its not just ergonomics; its also the issue that if you solely use generics to express this, then none of your code gets compiled until it ends up in the final application.

- acrichto: I assume all of the above traits would go in liballoc and reexport in libstd?
- nmatsakis: Think so.
- acrichto: I don't like that liballoc defines both the interface and the allocator. libcollections should not know that there's a default allocator. But we can deal with that later.
- brson: One problem is that porters have to hack up liballoc if jemalloc doesn't work for them.
- acrichto: Instead of a rexport, if we do a pub type, we could have libcollections not assume a default allocator, but then reexport them in libstd to have the default type param tied to the default allocator
- acrichto: But I think it's orthogonal to the above
- nmatsakis: Should be able to change this later on, since libcollections is unstable.
