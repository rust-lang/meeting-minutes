# Higher-ranked lifetimes/trait bounds

RELEVANT BLOG POST:
- http://smallcultfollowing.com/babysteps/blog/2013/11/04/intermingled-parameter-lists/

```
<'a> FnOnce(x: &'a int) -> &'a uint
```

- pcwalton: How should this be implemented?
- nmatsakis: It's similar to the early/late bound distinction. 
- pnkfelix: Is there a syntactic ambiguity in the below?

```
trait FnOnce {
    type ARG;
    type RETURN;
}

impl<'a> FnOnce(x: &'a int) -> &'a uint for SomeType {
    
}

<'a> FnOnce(x: &'a int) -> &'a uint
```

- nmatsakis: wrap TraitRef into HigherRankTraitRef
- nmatsakis: Idea is: an impl satisfies a higher-ranked trait reference if the 'a does not appear in the Self type or any of the other input types.
- nmatsakis: The way that early/late bound works is:
- nmatsakis: The idea is that early-bound lifetimes have to be given at any reference of the function.


```
fn foo<'a, 'b, T:'a>(...) { // early bound life-time: 'a, late-bound lifetime: 'b
}

let f: <'b> fn(...)
```

- nmatsakis: Taking a step back, if you have a set of type params T, as soon as you reference that functions, conceptually you have to instantiate T (for monomorphization among other reasons.)
- nmatsakis: But if there are lifetimes, referencing foo may incur some obligations about the lifetime, but that means we need some value for the lifetime. You have to specify the lifetime at the same time as the type -- so it's early bound, meaning that we have to instantiate it immediately. We have to have a value for it so we can process the type bounds.
- nmatsakis: On the other hand, a lifetime that does not appear in any trait bounds, the type is late bound, meaning that I don't have to instantiate it; I move it into a higher-ranked type.
- nmatsakis: That's how it works now. I think we can do the same thing for impls.
- nmatsakis: If my impl is given a particular self type, that implies a particular lifetime (there's a one-to-one relation). But in other cases, I may have a lifetime param that is not tied to Self. This works just like early/late bound, and for late bound cases we can get a higher-ranked type.

```
fn foo<T>(...)
let f = foo::<U>;
```

```
// early bound lifetime
fn foo<'a,T: SomeBound<'a>>(...) {
}
let f = foo::<'0, U>; // incurs the obligation that U : SomeBound<'0>
```

```
// late bound lifetime
fn foo<'a>(x: &'a Foo) -> &'a uint { ... }
let f = foo; // <'a> fn(&'a foo) -> &'a uint
```

```
impl<'a> SomeTrait<'a> for &'a uint { ... }
 --> SomeTrait<'a>
 
impl<'a> SomeOtherTrait<'a> for uint { ... } // no matter what lifetime 'a I am given, it works
  --> <'a> SomeOtherTrait<'a>
```

- pcwalton: What about the site of use? A function that takes a higher-ranked trait?
-  nmatsakis: It's only in vtable matching where we care about this: I  have this higher-ranked trait reference, and I have to find an impl that  matches it.
- nmatsakis: Probably I should do this, since I'm trying to land a rewrite of the relevant code anyway.
- nmatsakis: But anyway, it'd be during impl matching that we'd check the late-bound lifetimes
- pcwalton: Maybe I can start with an unsound version (that basically ignores this) to unblock people, and then we can fix it up afterwards
- nmatsakis: Still have to think about applications, where you need to fill in with fresh lifetimes
- pcwalton: So ty_param needs a lifetime?
- nmatsakis: No (though it does in my branch). The trait bounds have lifetimes
- pcwalton: I thought higher-ranked means you could call with many lifetimes.
- nmatsakis: It's not the lifetime of the reference to the function:

fn foo<T: <'a> FnOnce(&'a uint)>(t: T) {
    t.call() // instantiate 'a with '0, and then we know that T : FnOnce(&'0 uint)
}

- nmatsakis: We would say: call method is implemented by the FnOnce trait, so we'll replace 'a by a fresh lifetime
- pcwalton: So we don't have to carry it around throughout the function -- only at the moment you call
- nmatsakis: There is no borrowed content in T with that lifetime -- that's why it's unconstrained
- pcwalton: I'll do the syntax.
- nmatsakis: The subtyping with trait references should just be a port of what's there already.


