# Virtual structs

https://github.com/rust-lang/rfcs/pull/142

- nrc: There have been many proposals; maybe 2-3 reasonable alternatives
- nrc: One is: let's not try to be orthogonal; let's just do classes (RFC PR #5)
- nrc: The problem with that approach is it's unclear where to use enums and where to use virtual structs. It's simple, but not very Rust-y
- nrc: There's another RFC that uses unsizedness to give you the virtual-ness of struct objects, but then adds fields to traits to give you the efficient use of fields
- nrc: Or there's RFC 142, which has unsized struct objects but does inheritance in a more Rust-y fashion by unifying structs/enums and making them nestable. That's a big language change, but brings us closer to Scala/Haskell, which just have one kind of data type mechanism. Maybe we'd have a single "data" keyword, or keep struct/enum etc. The nice thing about this proposal is it's mostly backwards compatible -- you can continue having struct and enum today, and they'll continue working.
- nmatsakis: Are they truly unified? To me, there are two fundamentally different concepts. Today, enums are as big as the biggest of their variants -- basically by value, which is really useful for things like Option and others. "Virtual structs" (not the right name) -- the use case for DOM etc -- is more about, you will make an allocation that has a single type/representation
- nrc: There is this key difference. But other than the sizedness, going from struct to enum is just using a different keyword. Gives you a different runtime representation, though
- pnkfelix: Don't you have to use them differently?
- nrc: Yes. You could just have one keyword that does both (but has a way to specify sizedness) or have two keywords
- nmatsakis: I think a lot of the controversy is connected with trait objects/OOP modeling. This feature we're talking about, is not really a simpler trait object -- more like a more powerful enum.
- nmatsakis: Today enum has some serious limitations -- they have to be as big as the biggest variants, which is a problem for ASTs. You can work around it with indirection, but then you have to do more allocation etc.
- nmatsakis: We had some ideas about how to make this happen magically, but now it seems better to be explicit
- nmatsakis: Another problem is the inability to talk about sub-enums/refinement types. In my code, I make every variant have an associated struct, so that I can talk directly about that case.
- pcwalton: I do this in Servo. I have a pattern where each variant is a single word -- a box of the structure. You can pass these around, up and down-cast.
- nmatsakis: I do that too, usually without the box
- pcwalton: The nice thing about the box is that passing it around becomes quite cheap, and it deals with the variant size issue
- nmatsakis: This is like Java's representation, but pushing the fat pointer out rather than in. If I have multiple arguments in the variant, I'll almost always make a struct
- nrc: This just formalizes that pattern, basically. Beefed up enums, rather than simplified traits
- nmatsakis: We need to work out the design, but I think this is a better perspective. Trait objects are for open extensibility, enums are for closed variants
- pcwalton: Servo doesn't care about open/closedness -- you're not going to add new DOM types
- zwarich: The RFC limits extension to things in the same module/submodule. In Servo, we'd prefer crate-wide extension
- nmatsakis: Is this what we want out of this feature?
- pcwalton: I need to look at the details, but I think it's plausible
- nmatsakis: The fat pointer RFC, for example, seemed to have a different philosophy behind it
- pcwalton: I like the fat pointer proposal; I guess I"m neutral. I want something elegant, and I don't much care how we get there. Starting with enums or starting with traits could both be workable -- but note that Servo doesn't care about the open/closed distinction
- jack: My only comment is: it's fine to think of it that way, but the motivations you listed earlier about enums aren't in the RFC
- nmatsakis: Yes, we'll probably open a new RFC with the outcome
- brson: Do Niko's motivations match yours, Jack?
- nmatsakis: I think they match Rust's motivations
- zwarich: We don't want DOM objects to be bloated, and we want efficient ... through vtables
- nmatsakis: One motivation for making things closed is to make safe downcasting efficient. The openness makes RTTI expensive, for example
- pcwalton: Maybe Servo does want it to be closed
- nmatsakis: It'll be faster
- zwarich: Single inheritance is also more efficient
- nmatsakis: The motivation for the module restriction was privacy, and how to initialize super trait fields. When I talked to jdm about this proposal, that was his biggest concern. Here's the problem:

```
struct Base { 
    f: uint,
    ...
}

impl Base {
    fn new() -> Base {
        Base { f: 0 }
    }
}

struct Sub : Base { g: uint }

// What you *want* to write:
impl Sub {
    fn new() -> Sub {
        Sub { g: 0, .. Base::new() }
    }
}
```

- nmatsakis: w/ privacy, the private field is only visible in the mod that defs the base type
   if you want to init the subtype, the only way is by specifying all the base fields at the same time
   requiring you to be in the same module
- nmatsakis: jdm objected to having to repeat the fields period, because it's repetetive.
  naive thing you want to write is a constructor, but if the base is unsized it can't be returned.
- nmatsakis: We could make a context-dependent rule that handles this case, but it feels like a hack
- brson: We could always add that capability later, right?
- nmatsakis: Yes
- jack: laying out types in a tree of modules is wierd for servo because of types like 'EventTarget' that are (referenced from multiple nodes?). (here is jdm's conversation: http://logs.glob.uno/?c=mozilla%23servo&s=8+Aug+2014&e=8+Aug+2014&h=EventTarget#c99843 )
- pcwalton: What happens if we don't have the "virtual" keyword? You'd still allow structs to inherit other structs. (Looking at jdm's example)
- nrc: As proposed, extending something that's not virtual is an error
- pcwalton: Why, because there's no vtable?
- pcwalton: My primary concern (and the community's too, I think), is that it's a *lot* of machinery. Can we do this incrementally, and see if Servo can get by with a smaller design?
- pcwalton: The core of the RFC is that you can extend structs, and write virtual and override on methods, and it does what you expect. Can we do just that?
- nmatsakis: We had the base structs be unsized. I think the main motivator is to prevent slicing; otherwise, you could have a pointer to the base type ... [moving to whiteboard]
- nrc: It's whether you're a struct or enum that tells you whether you're sized -- not the virtual keyword. The virtual keyword is mainly used to emulate "pure virtual" in the C++ sense
- pcwalton: I wonder whether Servo really needs that
- nrc: It's just a "nice to have"
- nrc: Leaf structs are sized, non-leaf structs are unsized. You can tell which is which because it's closed
- pcwalton: What decides whether there's a vtable?
- nmatsakis: Whether you have virtual fns? Or maybe there's always a vtable, but it might be 0 sized
- nrc: Basically, you assume that you have a vtable, and in certain cases you can optimize it away
- pcwalton: I'm glad that we took the RFC about non-determined struct representation. You can use repr(C) to force a layout, and that will then disallow the virtual struct stuff
- nrc: A struct pointer points to the vtable. When you have a struct object, you have to use a negative offset
- nrc: You can only pass leaf structs by value
- pcwalton: Why do we have to unify enums and structs at all to do this?
- nmatsakis: I don't think we need to
- pcwalton: Just make structs support virtual/override. See how far Servo gets with that. We could pitch that to the community, and it seems much more agreeable
- nrc: I pitched that a long time ago, and it got more negative feedback
- pnkfelix: We proposed this at the last workweek; I think the problem was that we claimed it solved all the problems. Here, we just need to show that there's a path forward to solve other problems
- nmatsakis: What does the unification make possible that is truly useful?
- nrc: It doesn't deal with the problem you mentioned earlier about making enum variants use structs
- nmatsakis: I'd have to box them
- pcwalton: How do you pattern match against these?
- nrc: Just like enums: matching is basically how you to downcasting
- pcwalton: That's nice
- nrc: We can satisfy all the constraints with this limited design, but it feels bolted on. The unification idea is more about having a nicer language
- nmatsakis: If you can group your enums, the refinement case gets better. So there are expressiveness advantages
- pcwalton: My gut tells me it's easier to start with just the virtual struct
- nrc: struct enum variants fall out naturally from this plan
- nmatsakis: I feel like tuple structs and enum variants have weird interactions. A tuple must be a leaf, right?
- nrc: I think so
- nmatsakis: if enums are sized, is there a soundness problem? you can't upcast an &mut enum pointer to an &mut base. You could overwrite it and then violate the type system. But &mut is not really the right tool for this use case [discussion at whiteboard]
- nmatsakis: What we really want is something that lets you mutate the inteior fields, but not the thing as a whole -- so you can't change the type
- nmatsakis: So enums and structs give you different capabilities, and with different costs. It's a little disappointing, because you might want to match and then mutate some of the fields, and there's no way to say that -- &mut is too powerful
- nmatsakis: Put differently, this is why non-leaf structs must be unsized -- otherwise we lose all the nice capabilities
- nmatsakis: This is exactly what failed when we said we'd only have enums and allocate the memory at just the right size -- we'd have situations like this.
- nmatsakis: Also want to talk about generic type parameters and trait matching, how that interacts. In nrc's proposal the type parameters of the base were all inherited?
- nrc: I think I changed that so that you specify them, the same way you do with Java classes
- nmatsakis: The nested enum/struct sugar would assume they all have the same type params -- if you use that sugar, you cannot vary the number of type parameters for the variants
- nrc: I didn't specify any extra sugar for that. 
- nmatsakis: You want to be able to write

```
enum Option<T> {
    Some(T), None
}
```

without having to duplicate the type parameter.

- nrc: Yeah, that's in the proposal; you can omit the parameters in the nested variants
- pcwalton: What about RFC PR 11?
- nmatsakis: I think my concern was: (1) more complex, (2) a lot of the efficiency we can get with the more specialized proposal, we lose
[discussion of comments/proposals in https://github.com/rust-lang/rust/issues/9912]
...
- dherman: You can always say: "It's very fashionable these days to say composition is better than inheritance. But here are the cases where inheritance really is better, and our story for it"
- pcwalton: We could make it look more like Go 
- nrc: I have a syntax like that in the RFC
- nmatsakis: There are some slicing-like hazards we need to be aware of. Because of the unsized criteria, we cannot overwrite a base pointer with a different type -- which is good, since it's custom allocated with the right size. Nevermind -- no hazard
- nmatsakis: Today, method resolution involves a list of all the receivers; we check if the actual receiver is a subtype of what the trait would expect and could consider coercion
- nmatsakis: But here, there's no subtyping relationship -- that would be wrong, since by value they're not compatible. But there is a coercion relationship from &Sub to &Base. Basically, this determines where in the typechecker we will infer it. But I'm saying that if in a trait you take &self and you impl the trait on Base, the current rules will allow it to be callable on Sub. My proposed rules might be problematic, but I think they need to be revised anywa
- pcwalton: What if we split this into two proposals. First, "composition" a la Go:

```
struct Sub {
    Base;
}

where &Sub -> &Base coerces
```

- pcwalton: Separately, we propose virtual methods, and they complement the earlier proposal.
- dherman: I think a single proposal is better: explain why it's important and how it fits into Rust's overall design. We need to understand the larger story, where Servo is one example of a larger class of programs that needs this design
- nmatsakis: We're filling in gaps of today's enums
...
- nmatsakis: So I guess the big choice is between "minimalist" or "maximalist" design (both enum/struct). Probably we'd implement the minimalist one first regardless
- nrc: There is a backcompat hazard: enum variants with the same name as the type
- nmatsakis: I've done that a lot, because I'm modeling this proposal -- enum variants with the same name as the struct. But that can be changed
- nmatsakis: Still thinking about interaction with traits. If you implement a trait on the base type -- the trait will have to take Self as unsized. I'm worried about generic functions that take Option<Sub> and you want to use that with a function Option<T> where T: Trait and Base: Trait but not Sub: Trait.
- nmatsakis: Maybe you just have to impl the trait for &Base. I'm just worried about funny interactions, especially around subtyping.

```
struct Base { }
struct Sub : Base { }

impl Foo for Base { fn foo(&self) { ... } }

trait Foo for Sized? { fn foo(&self); }
fn do_something<T>(x: &Option<T>)
  where T : Foo
{
    match *x {
        Some(ref y) => y.foo()
        None => { }
    }
}

////

let x: Option<Sub> = ...;

// this call does not work
do_something(&x); // Error: type Sub does not implement Base

// but this code inlined does work
{
    match x {
        Some(ref y) => y.foo() // works
        None => { }
    }
}

///

trait Bar  { fn bar(&mut self); }

/// workaround

fn do_something<T:Foo>(x: Option<&T>) { ... }
do_something(x.as_ref());

```

- nmatsakis: Currently traits are invariant wrt Self. If we changed that and had subtyping, this might work. But there are still &mut problems, since that would force invariance.
- nmatsakis: Still some weird corners; I don't know if it's fixable without changing how &mut works

