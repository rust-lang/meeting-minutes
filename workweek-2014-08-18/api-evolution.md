# API Evolution

- aturon: there have been a number of times that we have talked about hazards for growing or changing APIs over time
- aturon: example is: if you add a method, you break code all over
- aturon: I thought it would be a good idea to define systematically the ways we think APIs ought to be able to evolve and make sure we have a good story
- aturon: open-ended enumerations came up in this work week, concrete idea
- aturon: for some of the other things there may not be a good sol'n, you may just have to swallow some minor breakage, I don't have much prepared
- acrichto: two concrete things I'm worried about: adding items breaks imports, adding methods breaks traits
- acrichto: only think you can TRULY add backwards compat is a crate
- acrichto: a different treatment of globs would solve adding items
- acrichto: but adding a method is worrisome
- acrichto: enums are worrisome, e.g., I shouldn't be able to match over all I/O errors
- aturon: for that last case we have workarounds today, e.g. trait objects, but way less convenient
- nmatsakis: may be good to enumerate list of things one might do

# Things people might want to do and how they can plausibly defend against them

- Add an item:
  - fns
  - traits
  - enums
  - structs
  - modules
  - pub use
  - use
- Add methods to a trait
  - Defaulted
  - Not defaulted
- Add variants to an enum
- Add fields to a struct
- Add a trait impl
- Add an inherent method
- Add a field to an enum variant
- Renaming a method
- Adding an argument to a method?
- Changing type of method argument to be more general
- Changing between generics and trait objects
- Converting inherent methods to trait methods
- Moving modules
- Add a new type parameter to a generics list
  - Defaulted
  - Non defaulted

# Today

- Adding a private use is ok
- Adding public things is not ok due to globs, which are feature-gated
- Note: proposed resolution rules for glob helps here

- Adding a non-default method to a trait: hopeless
- Adding a default method:
  - implements with other traits that have some method
    - AND are implemented for same types

- aturon: in terms of what stability attributes mean, some of these changes can cause breakage but only in limited circumstances, we should clarify if that is an OK thing to do with a stable API
- brson: we can of course use data (check with CI system)
- nmatsakis: we can use data to minimize breakage but we should give ourselves the right to add new methods and stuff

# Inherent methods and trait methods

- nmatsakis: Today, we first search for inherent methods via derefing, and only if we find nothing do we consider traits
- nmatsakis: We changed to that semantics so that we could implement trait on trait objects -- otherwise you'd not have any way to refer to the methods.
- nmatsakis: They have some appeal, in that you have some control, but it may not be the best.
- acrichto: so if you are adding a method to a trait you can get an ambig error due to conflicting with another trait
- brson: and you can use UFCS to fix it, and that is easy
- acrichto: what can we do to make that automatic?
- brson: maybe rustfix could kind of figure that out
- nmatsakis: would have to reason about diff
- acrichto: so we're basically accepting that adding a trait method is inherently broken but probably ok
- nmatsakis: I think that's ok but not unusual
- pnkfelix: what about adding an inherent method ...
- acrichto: not yet :)

# Adding variants to an enum

- acrichto: I would never ever considing extending Option but I would want to extend IOError
- acrichto: private enum variants were a useful way to make it so that you can never exhaustively match
- nmatsakis: I'm not sure that it was the most...direct way

```
enum Foo {
    bar,
    ..
}
```

- pcwalton: you can use newtyped constants (statics and ints), then they can never be exhaustive
- brson: can we make IOError opaque so that it's not so ugly
- acrichto: today with an enum as integral
- brson: not with IOError, it has a payload
- acrichto: oh yeah so that pattern doesn't work anymore
- nmatsakis: shew, I was having a hard time arguing against that
- brson: another option would be to make `Other(Box<Any>)`
- nmatsakis: what about adding `..` which makes all matches be considered non-exhaustive
- brson: we recently moved all failure from match, right?
- nmatsakis: you'd be forced to write an `_`, matches still are infallible
- aturon: so proposal on table would force you to add a wildcard match but is not extensible in the sense of cross-crate
- nmatsakis: I don't want that
- aturon: it might be useful if you wanted to have an open ADT for exceptions...
- nmatsakis: I prefer trait objects for that problem
- acrichto: I think this is in the bucket of "this will never break..."
- aturon: this is a way of making a guarantee, saying you can't rely on these not growing in the future
- acrichto: as crate author, it should be considered exhaustive
- nmatsakis: oh yeah I wanted to add that
- acrichto: we'd have to special case deriving to handle this case
- nmatsakis: not with the rule that you don't need it in the defining crate
- acrichto: I'm not convinced it's a good change, but it's non-invasive and backwards compat
- brson: backwards compat?
- acrichto: we can certianly add it after 1.0, though of course things wouldn't be able to use it
- aturon: what makes you ambivalent?
- acrichto: I consider adding item/method more dangerous, not sure how big a problem this is
- aturon: what is alternative? just say that we call it stable and say that we are allowed to add new variants?
- pnkfelix: argument you made before was that exhaustively matching against an I/O error is silly and I have to agree, but what about other example like AST?
- nmatsakis: I have been assuming that if we exported a stable AST API it'd be based on trait objects and awful, so that we can add new kinds of AST nodes, this gives us a better pattern
- acrichto: not sure how widely this applies
- nmatsakis: I feel like it will come up but would have to apply to virtual structs too perhaps

# Adding fields to a struct

- acrichto: adding a private field is ok 
- nmatsakis: I think there is a bug in autoderef that private fields shadow
- huon: adding a private field can affect size of type
- nmatsakis: we're not considering ABI compat for sure...
- aturon: adding private field when there wasn't one before IS a significant change
- acrichto: yes
- brson: so what do you do if you want to add a private field...?
- aturon: I think you just prefer methods
- acrichto: sometihng about `..`
- brson: using encapsulating like a method feels like
- nmatsakis: data we looked at suggested fields were all private or all public, I feel like there is not a big problem
- acrichto: it is a problem
- nmatsakis: but not a big problem
- brson: can we think of value-type structs that are at risk?
- acrichto: almost ever public facing API I can think of are all public fields
- aturon: IOError for example is all public, but we're prob going to encapsulate and it's not likely to change. Encapsulation is the right thing.

# Adding trait implementation

- nmatsakis: obviously can conflict with new ambig methods
- nmatsakis: we already said this was ok when adding new methods
- acrichto: say I forgot to implement `clone`...I can't purely add an impl of `clone` after the fact
- aturon: what do you mean by purely? you might break something?
- acrichto: yes
- acrichto: this seems like a second case where STABLE means actually it might break
- brson: strategies to mitigate potential breakage? recent refactoring was horrible due to 2 traits implementing to_str. Some patterns may reduce future conflicts...
- acrichto: avoid nice method names
- aturon: the more central concepts we can get into std (e.g. clone) that we are encouraging people to implement up front the better
- aturon: maybe that relates to a very different question: what does in stdlib?
- nmatsakis: sometihng to enqueue for later is a list of traits you really ought to implement
- nmatsakis: feel like the last time acrichto shot down my proposal with lots of good reasons that the list was not good

# Adding an inherent method

- acrichto: picks a new method if there is already an existing one
- pnkfelix: worries me
- nmatsakis: argument and return types must also match
- acrichto: it is a good point, scarier
- pnkfelix: could just be that adding inherent methods is not permitted in a stable API
- nmatsakis: I just think nobody is going to do it
- brson: this is mostly about stdlib
- acrichto: resolution cargo performs does expect that semver is meaningful
- brson: why does cargo care?
- nmatsakis: has to know what to upgrade
- acrichto: i.e., if you need library with version 1.3 and 1.7, we'll just take 1.7
- nmatsakis: I think we should strongly consider making any conflict an error, rather than the trump rules
- nmatsakis: ... re-explains this idea that `Deref<T>` can only look for trait methods when fully deref'd ...
- ... some questions ...
- pnkfelix: if I were to use UFCS to ask for a particular trait would you stlil autoderef?
- nmatsakis: no UFCS doesn't autoderef, just plain fn call notation

... lots of discussion about what would happen if inherent methods conflicted with trait methods

- aturon: what are reasons to have inherent methods trump?
- acrichto: only hard reason is b/c we don't have DST
- aturon: seems clear that we should remove that rule
- acrichto: in which case you are allowed to add inherent methods to stable things, may yield an error

# Adding a field to an enum variant

- acrichto: same as struct fields, but they are all public for public enums, so you just can't do it
- brson: just have to make it a new type
- nmatsakis: make a major release and call it a day

# Renaming a method

- brson: obvious
- acrichto: deprecation?
- brson: yeah leave old one in place and mark it as deprecated

# Adding an argument to a method

- aturon: if we had default arguments...but we don't.
- aturon: when we want to grow our APIs we can add this feature.
- acrichto: so no not on a public method.

# Changing type of method argument to be more general

- aturon: today you are safe writing against a concrete type,
- aturon: tomorrow you can introduce a trait bound and match against T
- acrichto: most of the time that works but ... example?
- aturon: you're taking a slice today but tomorrow you want an iterable
- aturon: should be fine since slices impl iterable
- acrichto: ah, yes, I think that's fine
- brson: can't take value of the function
- nmatsakis: what?
-  brson: can I show you an example

```
fn foo(a: &str) { ... }
=>
fn foo<T: Bar>(a: T) { ... }

let x: fn(&str) = foo;

let x = foo;
```

- nmatsakis: it is possible that there would be inference failures in some scenarios
- nmatsakis: seems to be that this falls under the category of "minor breakage", similar to adding methods
- nmatsakis: I would like to be able to say that we give ourselves the right to do things where the old types would continue to work, if fully annotated, but of course we'd take into account the real-world impacts
- acrichto: I think aturon is right that we're going to wind up doing this for things like `&[T]` to `Iterable`
- aturon: one related thing is when there is a type and there is a trait and they are in separate crates ...
- nmatsakis: ... yeah, but it's not related to this, right?
- acrichto: so we just want to stomach that this could cause subtle inference failures but we expect it to be sufficiently rare
- nmatsakis: yes I do
- aturon: agreed

# changing between generics and trait objects

```
// not possible
pub fn foo<T:Bar>(t: T) {...}
    ==> fn foo(t: Bar) { ... } // IMPOSSIBLE

// BUT you can convert the `foo` above to this:
pub fn foo<T:Bar>(t: T) { foo1(box t) }
fn foo1(b: Box<Bar>) { ... } // here b is borrowed, not owned

// generally ok under DST, presuming that `Bar : Bar`
// --> depends on whether all objects implement their trait or not
fn foo(x: Box<Bar>) ==> fn foo<T:Bar>(x: Box<T>)
```

# inherent to trait methods

- nmatsakis: clearly breaking because you must import trait
- nmatsakis: however, you can leave a tombstone -- oh, no, you can't, not if we change rules to be ambig
- nmatsakis: what we give up here is ability to extract out common inherent methods into a trait without brekaing caller, or using distinct method names
- aturon: seems like trumping is worse

# moving modules

- brson: same as renaming anything else, leave a `pub use`

# adding new type parameter to generics list

- nmatsakis: if you have a default it's ok modulo how defaults and inference interact
- aturon: is there a rule like if you provide any you must provide all?
- nmatsakis: I don't think so
- pnkfelix: but for methods it might seem relatively ok, we already argued above we're going to let you switch from non-generic to generic

```
// BEFORE:
fn foo<A,B>(...)
let x = foo::<int,int>()

// AFTER:
fn foo<A,B,C>(...)
let x = foo::<int,int> // ERROR
```

- aturon: what happens if you specify an empty list?
- nmatsakis: I think code just considers empty list and `::<>` completely equivalent
- aturon: I guess question is whether we think an error like that is a problem
- pnkfelix: Niko was arguing before that we are more willing to make inference fail, this is fully elaborated
- nmatsakis: yes though I inconsistenty feel like this is ok too
- aturon: is there a fundamental reason you can't provide a partial list of parameters?
- nmatsakis: no, just sanity check
- aturon: my gut feeling is that this doesn't seem worth worrying about
- aturon: other thing is ... trait methods?
- aturon: but I guess those are different
- nmatsakis: it boils down to the fact that people rarely specify type parameters
- aturon: except for collect
- nmatsakis: in cases where types can usually be inferred from argments it's ok, but collect/size-of not so good
- nmatsakis: I guess I justify it mysel by saying that it feels like just changing fiddly type annotations -- either adding, due to inference, or adjusting, due to elaboration -- feels ok

# Summary

- acrichto: I am deeply uncomfortable with globs
- acrichto: but otherwise ok
- acrichto: we are admitting that it's not 100% backwards compat but everything that's still reasonable is reasonable
- aturon: quite a lot of work on APIs without officially breaking
- aturon: two questions requiring resolution
  - extensible enum
  - exact rules for inherent methods trumping and interaction with autoderef
- nmatsakis: so jakub- has been doing awesome on matching and so on, maybe he has interest
- brson: a lot of feelings here about what is and isn't breakage but no data
- brson: it would be nice if we could capture metrics to measure this somehow
- acrichto: this is where I feel like releases before 1.0 would be very useful
- acrichto: very tough to measure
- aturon: related, someone should write up more formally, connect to stability attributes
- brson: I think plan is to create global action items
- brson: I am worried that we've got too many
- nmatsakis: we may have to pare back
- brson: More worried we're going to forget some of them
- pnkfelix: I want to say... the cargo issue is more severe than I was thinking. I realize now that due to cargo it may be that upgrading rust causes some 3rd party library to stop compiling and I have to go fix it.
- acrichto: a few things we can do about that. We obviously can't control if other people follow semvar, but...
- brson: if we claim we're doing semver, people will say you're not doing any form of ABI compat
- brson: we're gonna break stuff
- acrichto: ...ties into cargo's policy on crates. We might want to make it less eager to coallesce.
- nmatsakis: we'll have to see how it goes, but we can't tie our hands too much. APIs grow.
- acrichto: I've heard talk about a tool to test this. Hash and compare API.
- aturon: reasonable place to start is with rustdoc metadata
- nmatsakis: wycats was talking about this as part of cargo
- acrichto: would go a long way towards helping 
- aturon: it'd be really useful just for std
- acrichto: oh so many things I want for std
- brson: I'm very enthusiastic about this idea of cargo checking semver rules. Maybe we can find an intern who would be interested. Doesn't seem so hard given that we have fixed rules.
- acrichto: we'll have data avail if we have a tool that is easy
- brson: wait so we could catalog what has changed and then cargo update could automatically patch it
- *general astonishment at what a genius idea this is*


















